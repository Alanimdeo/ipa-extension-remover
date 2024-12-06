<script lang="ts">
  import { FileDropzone } from "@skeletonlabs/skeleton";
  import { SlideToggle } from "@skeletonlabs/skeleton";
  import { ProgressBar } from "@skeletonlabs/skeleton";

  import * as zip from "@zip.js/zip.js";
  import { parse as parsePlist } from "@plist/parse";
  import type { Dictionary, Value as PlistValue } from "@plist/common";
  import { convert as cgbiToPng } from "cgbi";

  import ArrowLeft from "./assets/arrow-left.svg";
  import FileUpload from "./assets/file-upload.svg";

  const metadataRegex = /((:?\/)?Payload\/[^/]+\.app\/)Info\.plist/;
  const appIconRegex = (appIconName: string) =>
    new RegExp(`Payload\\/[^/]+\\.app\\/${appIconName}(@\\d+x)?\\.[^/]+`);
  const extensionRegex = /Payload\/[^/]+\.app\/PlugIns\/(?:([^/]+)\.appex)\//;

  let files: FileList | undefined = $state(undefined);

  let abortController: AbortController | null = $state(null);

  let ipa: zip.Entry[] | null = $state(null);
  let metadata: Dictionary | null = $state(null);
  let appIcon: string | null = $state(null);
  let extensions: zip.Entry[] = $state([]);
  let extensionsToKeep: {
    [extension: string]: boolean;
  } = $state({});
  let saving = $state(false);
  let progress = $state(0);
  let progressMax = $state(0);
  let progressText = $state("Idle");

  function clearIPA() {
    if (abortController) {
      abortController.abort();
    }
    files = undefined;
    ipa = null;
    metadata = null;
    appIcon = null;
    extensions = [];
    extensionsToKeep = {};
    saving = false;
    progress = 0;
    progressMax = 0;
    progressText = "Idle";
  }

  async function loadIPA() {
    if (!files || files.length === 0) throw new Error("No file selected");
    const file = files.item(0);
    if (!file) throw new Error("No file selected");

    const reader = new zip.ZipReader(new zip.BlobReader(file));
    const entries = await reader.getEntries();
    ipa = entries;
    extensions = entries.filter(
      (entry) => entry.directory && entry.filename.endsWith(".appex/")
    );
    extensionsToKeep = Object.fromEntries(
      extensions.map((extension) => [extension.filename, true])
    );
    metadata = await getMetadata();
    if (metadata?.CFBundleIcons) {
      const cfBundleIcons = metadata.CFBundleIcons as Dictionary;
      const primaryIcon = cfBundleIcons.CFBundlePrimaryIcon as Dictionary;
      const iconFiles = primaryIcon.CFBundleIconFiles as PlistValue[];
      const iconFilename = iconFiles[0] as string;
      const regex = appIconRegex(iconFilename);
      const iconEntry = ipa.find((entry) => regex.test(entry.filename));
      if (iconEntry && iconEntry.getData) {
        const iconData = await iconEntry.getData(new zip.Uint8ArrayWriter());
        const decodedPng = await cgbiToPng(iconData);
        appIcon = URL.createObjectURL(
          new Blob([decodedPng], { type: "image/png" })
        );
      }
    }
  }

  async function getMetadata(): Promise<Dictionary | null> {
    if (!ipa) return null;
    const metadataEntry = ipa.find((entry) =>
      metadataRegex.test(entry.filename)
    );
    if (!metadataEntry || !metadataEntry.getData) return null;
    const metadataBlob = await metadataEntry.getData(new zip.BlobWriter());
    try {
      const parsed = parsePlist(await metadataBlob.arrayBuffer());
      return parsed as Dictionary;
    } catch (error) {
      console.error(error);
      return null;
    }
  }

  function getExtensionName(path: string) {
    const match = path.match(extensionRegex);
    if (!match) return null;
    return match[1];
  }

  async function saveIPA() {
    if (abortController || saving) return;

    abortController = new AbortController();
    try {
      saving = true;
      await _saveIPA(abortController.signal);
    } catch (error) {
      console.error(error);
    } finally {
      abortController = null;
      saving = false;
    }
  }

  function _saveIPA(abortSignal: AbortSignal) {
    return new Promise<void>(async (resolve, reject) => {
      let aborted = false;

      abortSignal.addEventListener("abort", () => {
        aborted = true;
        progress = 0;
        progressMax = 0;
        reject(new Error("Aborted"));
      });

      progress = 0;
      if (!files || files.length === 0 || !ipa) {
        return reject(new Error("IPA not loaded"));
      }
      const file = files.item(0)!;
      if (Object.values(extensionsToKeep).every((e) => e)) {
        if (aborted) return;
        progressMax = 1;
        progress = 1;
        progressText = "Done!";
        saveBlob(file, file.name);
        return resolve();
      }
      progressText = "Filtering extensions";
      const extensionsToRemove = extensions
        .filter((extension) => !extensionsToKeep[extension.filename])
        .map((extension) => extension.filename);

      progressText = "Creating new IPA archive";
      const blobWriter = new zip.BlobWriter("application/zip");
      const zipWriter = new zip.ZipWriter(blobWriter);
      progressMax = Math.round(ipa.length * 1.1);
      for (const entry of ipa) {
        if (aborted) break;
        progress++;
        let skip = false;
        for (const extension of extensionsToRemove) {
          if (entry.filename.startsWith(extension)) {
            skip = true;
            break;
          }
        }
        if (skip) continue;
        if (entry.directory) {
          progressText = `Creating directory ${ellipsis(entry.filename.split("/").at(-2) || "", 15)}`;
          await zipWriter.add(entry.filename, undefined, { directory: true });
          continue;
        }
        if (entry.getData) {
          progressText = `Adding ${ellipsis(entry.filename.split("/").at(-1) || "", 30)}`;
          const data = await entry.getData(
            new zip.BlobWriter("application/octet-stream")
          );
          await zipWriter.add(entry.filename, new zip.BlobReader(data));
        }
      }
      if (!aborted) progressText = "Finalizing archive";
      await zipWriter.close();
      if (aborted) return;
      progressText = "Saving archive";
      saveBlob(await blobWriter.getData(), file.name);
      progress = progressMax;

      progressText = "Done!";
      resolve();
    });
  }

  function ellipsis(text: string, max: number) {
    if (text.length > max) {
      return text.slice(0, max - 3) + "...";
    } else {
      return text;
    }
  }

  function saveBlob(blob: Blob, filename: string) {
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = filename;
    a.click();
    URL.revokeObjectURL(url);
  }
</script>

<main>
  {#if !files}
    <FileDropzone
      name="files"
      bind:files
      accept=".ipa"
      onchange={loadIPA}
      disabled={saving}
    >
      <svelte:fragment slot="lead">
        <img src={FileUpload} alt="Upload icon" class="w-16 mx-auto" />
      </svelte:fragment>
      <svelte:fragment slot="message">
        <p class="px-10">
          <strong>Upload an .ipa file</strong> or drag and drop
        </p>
      </svelte:fragment>
    </FileDropzone>
  {:else if !ipa}
    <p>Analyzing IPA...</p>
  {:else}
    <div class="flex justify-end">
      <button class="text-left mb-2 mr-auto" onclick={clearIPA}>
        <img class="w-6 inline" src={ArrowLeft} alt="Back icon" />
        Back
      </button>
    </div>
    {#if metadata}
      <div class="card mb-2">
        <header class="card-header h3">Selected IPA</header>
        <section class="card-header w-fit p-4">
          <div class="flex gap-4">
            {#if appIcon}
              <img src={appIcon} alt="App icon" />
            {/if}
            <div>
              <h2 class="h2 text-left">
                {metadata.CFBundleDisplayName || metadata.CFBundleName}
              </h2>
              <p class="text-left">{metadata.CFBundleIdentifier}</p>
              <p class="text-left">
                {metadata.CFBundleShortVersionString ||
                  metadata.CFBundleVersion}
              </p>
            </div>
          </div>
        </section>
      </div>
    {/if}
    {#if extensions.length === 0}
      <h4 class="h4">This app has no extensions!</h4>
    {:else}
      <h4 class="h4 mb-2">Select extensions to keep!</h4>
      <div class="flex justify-center">
        <div class="text-left flex flex-col gap-1 w-fit">
          {#each extensions as extension (extension.filename)}
            <SlideToggle
              size="sm"
              active="bg-primary-500"
              name={extension.filename}
              checked={extensionsToKeep[extension.filename]}
              disabled={saving}
              onchange={() =>
                (extensionsToKeep[extension.filename] =
                  !extensionsToKeep[extension.filename])}
            >
              {getExtensionName(extension.filename)}
            </SlideToggle>
          {/each}
        </div>
      </div>

      <div class="w-full mt-2">
        <span class="text-xs">{progressText}</span>
        <ProgressBar class="w-full" value={progress} max={progressMax} />
      </div>
      <button
        class="w-full btn variant-filled-primary mt-3"
        onclick={saveIPA}
        disabled={saving}
      >
        Save
      </button>
    {/if}
  {/if}
</main>
