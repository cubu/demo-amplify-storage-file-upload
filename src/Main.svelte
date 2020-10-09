<script>
  import { Storage } from "aws-amplify";
  import {formatBytes, downloadBlob} from './utils.js'
  let loading = true;
  let folders = [];
  let progressPercent = null;
  let files = [];
  let res = null;
  let currentPath = "";
  function createFolder(e) {
    let newName = window.prompt("Name of New Folder");
    if (newName) {
      Storage.put(currentPath + newName, undefined)
        .then(() => {
          loading = true;
          loadFiles();
        })
        .catch((err) => console.log(err));
    }
  }

  
  async function loadFiles() {
    return Storage.list("fodder/")
      .then((result) => {
        res = processStorageList(result);
      })
      .catch((err) => console.log(err))
      .finally(() => {
        loading = false;
      });
  }

  function onSubmit(e) {
    const file = e.target.fileInput.files[0];
    Storage.put(currentPath + file.name, file, {
      contentType: file.type,
      progressCallback(progress) {
        progressPercent = progress.loaded / progress.total;
      },
    })
      .then(() => {
        progressPercent = null;
        loading = true;
        loadFiles();
      })
      .catch((err) => console.log(err));
  }

  $: {
    if (res) {
      let temp = currentPath
        .split("/")
        .reduce((acc, cv) => (cv ? acc[cv] : acc), res);
      // console.log({currentPath, temp})
      let _res = Object.entries(temp).filter(([LHS]) => LHS !== "__data");
      folders = _res
        .filter(
          ([LHS, RHS]) => console.log({ LHS, RHS }) || RHS.__data.size < 1
        )
        .map(([LHS]) => LHS);
      files = _res
        .filter(([LHS, RHS]) => RHS.__data.size > 0)
        .map(([LHS, RHS]) => RHS);
      // console.log({folders, files})
    }
  }

  function processStorageList(results) {
    const filesystem = {};
    // https://stackoverflow.com/questions/44759750/how-can-i-create-a-nested-object-representation-of-a-folder-structure
    const add = (source, target, item) => {
      const elements = source.split("/");
      const element = elements.shift();
      if (!element) return; // blank
      target[element] = target[element] || { __data: item }; // element;
      if (elements.length) {
        target[element] =
          typeof target[element] === "object" ? target[element] : {};
        add(elements.join("/"), target[element], item);
      }
    };

    results.forEach((item) => add(item.key, filesystem, item));
    return filesystem;
  }
  import { onMount } from "svelte";
  onMount(() => {
    loadFiles();
  });
</script>

<style>
</style>

<main>
  <!-- header -->
  <div class="bg-white">
    <div class="max-w-screen-xl mx-auto py-16 px-4 sm:py-24 sm:px-6 lg:px-8">
      <div class="text-center">
        <h1 class="text-base leading-6 font-semibold text-indigo-600 tracking-wide uppercase">Amplify Storage + Svelte</h1>
        <p class="mt-1 text-4xl leading-10 font-extrabold text-gray-900 sm:text-5xl sm:leading-none sm:tracking-tight lg:text-6xl">File Upload and Navigation Demo</p>
        <p class="max-w-xl mt-5 mx-auto text-xl leading-7 text-gray-500">Upload any file to AWS S3, and browse your files.</p>
      </div>
    </div>
  </div>

  <!-- body -->
  <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
    <!-- We've used 3xl here, but feel free to try other max-widths based on your needs -->
    <div class="max-w-3xl mx-auto">
      <div>Path: {currentPath || '/'}</div>

      <div class="flex flex-col">
        <div class="-my-2 overflow-x-auto sm:-mx-6 lg:-mx-8">
          <div class="py-2 align-middle inline-block min-w-full sm:px-6 lg:px-8">
            <div class="shadow overflow-hidden border-b border-gray-200 sm:rounded-lg">
              <table class="min-w-full divide-y divide-gray-200">
                <thead>
                  <tr>
                    <th class="px-6 py-3 bg-gray-50 text-left text-xs leading-4 font-medium text-gray-500 uppercase tracking-wider">
                      Name
                    </th>
                    <th class="px-6 py-3 bg-gray-50 text-left text-xs leading-4 font-medium text-gray-500 uppercase tracking-wider">
                      Last Modified
                    </th>
                    <th class="px-6 py-3 bg-gray-50 text-left text-xs leading-4 font-medium text-gray-500 uppercase tracking-wider">
                      Size
                    </th>
                    <th class="px-6 py-3 bg-gray-50"></th>
                  </tr>
                </thead>
                <tbody>
                  {#if currentPath}
                    <tr class="bg-white">
                      <td class="px-6 py-4 whitespace-no-wrap text-sm leading-5 font-medium text-teal-800">
                        <svg class="w-6 inline"  xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 14l2-2m0 0l2-2m-2 2l-2-2m2 2l2 2M3 12l6.414 6.414a2 2 0 001.414.586H19a2 2 0 002-2V7a2 2 0 00-2-2h-8.172a2 2 0 00-1.414.586L3 12z" />
                        </svg>
                        <button class="border-0" on:click={() => currentPath = currentPath.split('/').filter(Boolean).slice(0,-1).join('/') + '/'}>Back</button>
                      </td>
                    </tr>
                  {/if}
                  {#if loading}
                    <tr class="bg-white">
                      <td class="px-6 py-4 whitespace-no-wrap text-sm leading-5 font-medium text-gray-900">
                        🌀 Loading...
                      </td>
                    </tr>
                  {/if}      
                  {#each folders as folder, i}
                    <tr class="bg-white" class:bg-gray-50={i % 2}>
                      <td class="px-6 py-4 whitespace-no-wrap text-sm leading-5 font-medium text-gray-900">
                        <svg class="w-6 inline" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M3 7v10a2 2 0 002 2h14a2 2 0 002-2V9a2 2 0 00-2-2h-6l-2-2H5a2 2 0 00-2 2z" />
                        </svg>
                        <button class="border-0" on:click={() => currentPath += folder + '/'}>{folder}</button>
                      </td>
                      <td class="px-6 py-4 whitespace-no-wrap text-sm leading-5 text-gray-500">
                        {folder.__data ? folder.__data.lastModified.toString().slice(0,24) : ''}
                      </td>
                      <td class="px-6 py-4 whitespace-no-wrap text-sm leading-5 text-gray-500">
                        
                      </td>
                      <td class="px-6 py-4 whitespace-no-wrap text-right text-sm leading-5 font-medium">
                        <a href="#" class="text-indigo-600 hover:text-indigo-900">Delete</a>
                      </td>
                    </tr>
                  {/each}
                  <tr class="bg-white">
                    <td class="px-6 py-4 whitespace-no-wrap text-sm leading-5 font-medium text-teal-700">
                      <button class="border-0" on:click={createFolder}>
                        <svg class="w-6 inline" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 13h6m-3-3v6m-9 1V7a2 2 0 012-2h6l2 2h6a2 2 0 012 2v8a2 2 0 01-2 2H5a2 2 0 01-2-2z" />
                        </svg>
                        Create New Folder</button>
                    </td>
                  </tr>
                  <!-- Odd row -->

                  {#each files as file, i}
                    <tr class="bg-white" class:bg-gray-50={i % 2}>
                      <td class="px-6 py-4 whitespace-no-wrap text-sm leading-5 font-medium text-gray-900">
                        <svg class="w-6 inline"  xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M7 21h10a2 2 0 002-2V9.414a1 1 0 00-.293-.707l-5.414-5.414A1 1 0 0012.586 3H7a2 2 0 00-2 2v14a2 2 0 002 2z" />
                        </svg>
                        <button class="border-0" on:click={() => {
                            Storage.get(file.__data.key, { download: true })
                            .then(res => console.log({res}) || downloadBlob(res.Body, file.__data.key.slice(currentPath.length)))
                        }}>{file.__data.key.slice(currentPath.length)}</button>
                      </td>
                      <td class="px-6 py-4 whitespace-no-wrap text-sm leading-5 text-gray-500">
                        {file.__data.lastModified.toString().slice(0,24)}
                      </td>
                      <td class="px-6 py-4 whitespace-no-wrap text-sm leading-5 text-gray-500">
                        {formatBytes(file.__data.size)}
                      </td>
                      <td class="px-6 py-4 whitespace-no-wrap text-right text-sm leading-5 font-medium">
                        <a href="#" class="text-indigo-600 hover:text-indigo-900">Delete</a>
                      </td>
                    </tr>
                  {:else}
                    <tr class="bg-white" class:bg-gray-50={i % 2}>
                      <td class="px-6 py-4 whitespace-no-wrap text-sm leading-5 font-medium text-gray-900">
                        This folder is empty!
                      </td>
                      <td class="px-6 py-4 whitespace-no-wrap text-sm leading-5 text-gray-500">
                      </td>
                      <td class="px-6 py-4 whitespace-no-wrap text-sm leading-5 text-gray-500">
                      </td>
                      <td class="px-6 py-4 whitespace-no-wrap text-right text-sm leading-5 font-medium">
                      </td>
                    </tr>
                  {/each}

                </tbody>
              </table>
            </div>
          </div>
        </div>
      </div>


      <!-- submit form -->
      <div class="mt-8 sm:mx-auto sm:w-full sm:max-w-md">
        <div class="bg-white py-8 px-4 shadow sm:rounded-lg sm:px-10">
          <form on:submit|preventDefault={onSubmit}>
            <div>
              <label for="fileInput" class="sr-only block text-sm font-medium leading-5 text-gray-700">
                Choose a file to upload
              </label>
              <div class="mt-1 rounded-md shadow-sm">
                <input type="file" name="fileInput" id="fileInput" 
                required class="appearance-none block w-full px-3 py-2 border border-gray-300 rounded-md placeholder-gray-400 focus:outline-none focus:shadow-outline-blue focus:border-blue-300 transition duration-150 ease-in-out sm:text-sm sm:leading-5"
                />
              </div>
            </div>

            <div class="mt-6">
              <span class="block w-full rounded-md shadow-sm">
                <button type="submit" class="flex justify-center py-2 px-4 border border-transparent text-sm font-medium rounded-md text-white bg-indigo-600 hover:bg-indigo-500 focus:outline-none focus:border-indigo-700 focus:shadow-outline-indigo active:bg-indigo-700 transition duration-150 ease-in-out">
                  <svg class="w-6 inline"  xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 13h6m-3-3v6m5 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z" />
                  </svg>
                  Upload
                </button>
              </span>
              {#if progressPercent !== null}
                <label for="file">Uploading {Math.round(progressPercent * 100)}%...</label>
                <progress id="file" max={1} value={progressPercent}> {Math.round(progressPercent * 100)}% </progress>
              {/if}
            </div>
          </form>
        </div>
      </div>
    </div>
  </div>
</main>