<script>
    //imports
    import {getContext, onDestroy, onMount} from "svelte";

    //input variables
    export let field;
    export let label;
    export let onChange;
    export let maxFiles;
    export let acceptedFiles;

    //Getting budibase API
    const {styleable} = getContext("sdk");
    const component = getContext("component");
    const formContext = getContext("form");
    const formStepContext = getContext("form-step");
    const fieldGroupContext = getContext("field-group");
    const {notificationStore} = getContext("sdk");

    //Budibase field group setup
    let fieldApi;
    let fieldState;
    const formApi = formContext?.formApi;
    const labelPos = fieldGroupContext?.labelPosition || "above";

    $: formStep = formStepContext ? $formStepContext || 1 : 1;
    $: formField = formApi?.registerField(
        field,
        "text",
        '',
        false,
        null,
        formStep
    );
    $: unsubscribe = formField?.subscribe((value) => {
        fieldState = value?.fieldState;
        fieldApi = value?.fieldApi;
    });

    $: labelClass =
        labelPos === "above" ? "" : `spectrum-FieldLabel--${labelPos}`;

    onDestroy(() => {
        fieldApi?.deregister();
        unsubscribe?.();
    });

    let buttonID = Math.random().toString(16);

    let files = []; //List of files - in link format
    let archivedFiles = []; //Stored copy of files in case a process fails

    let browseFiles = []; //Files that are selected to be uploaded

    //Generates JSON from files and stores it in the field
    let syncFilesWithField = async () => {
        //Check if adding selected files will breach file length restriction.
        if (maxFiles && files.length + browseFiles.length > maxFiles) {
            notificationStore.actions.warning(
                "Too many files. You can only upload " +
                maxFiles +
                " files into this field."
            );
            browseFiles = [];
            document.getElementById(buttonID).value = null;
            return;
        }

        archivedFiles = [...files];

        try {
            files = files.concat(await Promise.all(
                Array.from(browseFiles)
                    .map(async (selectedFile) => ({
                        name: selectedFile.name,
                        type: selectedFile.type,
                        size: selectedFile.size,
                        link: (await readFileToLink(selectedFile)).target.result
                    }))
            ));

            browseFiles = [];
            document.getElementById(buttonID).value = null;
            let json = JSON.stringify(files);

            const changed = fieldApi.setValue(json);

            if (onChange && changed) {
                onChange({value: json});
            }

            archivedFiles = [...files];
        } catch (e) {
            notificationStore.actions.warning("Error reading files. Try again.");
            files = [...archivedFiles];
            browseFiles = [];
            document.getElementById(buttonID).value = null;
        }
    };

    //Reads a file and returns a promise that resolves to a link
    function readFileToLink(file) {
        return new Promise((resolve, reject) => {
            const fileReader = new FileReader();
            fileReader.onload = (e) => resolve(e);
            fileReader.onerror = (e) => reject(e);
            fileReader.readAsDataURL(file);
        });
    }

    //Everytime the browseFiles array changes, the files are processed.
    $: if (browseFiles.length > 0) {
        syncFilesWithField();
    }

    //Parses inputted files and saves them to the files array.
    onMount(() => {
        if (fieldState?.value) {
            let jsonValue = fieldState.value;

            files = JSON.parse(jsonValue);

            if (maxFiles && files.length > maxFiles) {
                files = files.slice(0, maxFiles);
                notificationStore.actions.warning(
                    "Too many files. Only the first " +
                    maxFiles +
                    " files in the field will be kept."
                );
            }
            archivedFiles = [...files];
        }
    });

    //Opens a file in a new tab.
    let onLinkClick = (file) => {
        const image = new Image();
        image.src = file.link;

        const w = window.open("");
        w.document.write(image.outerHTML);
    };
    let deleteButtonClick = (index) => {
        files.splice(index, 1);
        syncFilesWithField();
    };
</script>

<div class="spectrum-Form-item" use:styleable={$component.styles}>
    {#if !formContext}
        <div class="placeholder">Form components need to be wrapped in a form</div>
    {:else}
        <label
                class:hidden={!label}
                for={fieldState?.fieldId}
                class={`spectrum-FieldLabel spectrum-FieldLabel--sizeM spectrum-Form-itemLabel ${labelClass}`}
        >
            {label || " "}
        </label>
        <div class="spectrum-Form-itemField">
            {#if fieldState}
                <input
                        type="file"
                        id={buttonID}
                        style="display: none;"
                        multiple
                        accept={acceptedFiles}
                        bind:files={browseFiles}
                />
                <input
                        type="button"
                        value="Browse"
                        class="browse-button"
                        onclick="document.getElementById('{buttonID}').click();"
                />
                <div>
                    {#each files as file, i}
                        <div class="file-button-body">
                            <button
                                    class="file-button file-button-delete"
                                    on:click={() => deleteButtonClick(i)}>✖
                            </button
                            >
                            <button
                                    class="file-button file-button-link"
                                    on:click={() => onLinkClick(file)}
                            >
                                {file.name}
                            </button>
                        </div>
                    {/each}
                </div>
            {/if}
        </div>
    {/if}
</div>

<style>
    .placeholder {
        color: var(--spectrum-global-color-gray-600);
    }

    label {
        white-space: nowrap;
    }

    label.hidden {
        padding: 0;
    }

    .spectrum-Form-itemField {
        position: relative;
        width: 100%;
        background-color: white;
        padding: 10px;
        border-radius: 4px;
        border: 1px solid rgb(190, 190, 190);
    }

    .browse-button {
        position: relative;
        width: 100%;
        height: 50px;
        cursor: pointer;
        border: 1px solid rgb(190, 190, 190);
        border-radius: 4px;
        border-width: 1px;
        color: rgb(110, 110, 110);
        font-weight: bold;
        font-size: 16px;
    }

    .file-button-body {
        position: relative;
        width: 100%;
        height: 50px;
        cursor: pointer;
        border: 1px solid rgb(190, 190, 190);
        border-radius: 4px;
        border-width: 1px;
        color: rgb(110, 110, 110);
        font-weight: bold;
        font-size: 16px;
    }

    .file-button {
        border: 0;
        font: inherit;
        color: inherit;
    }

    .file-button-delete {
        position: absolute;
        width: 40px;
        height: 100%;
        left: 0;
        border: 0;
        background-color: rgb(151, 151, 151);
        color: rgb(168, 87, 84);
    }

    .file-button-link {
        background-color: rgb(226, 226, 226);
        position: absolute;
        width: calc(100% - 40px);
        height: 100%;
        left: 40px;
    }

    .error {
        color: var(
                --spectrum-semantic-negative-color-default,
                var(--spectrum-global-color-red-500)
        );
        font-size: var(--spectrum-global-dimension-font-size-75);
        margin-top: var(--spectrum-global-dimension-size-75);
    }

    .spectrum-FieldLabel--right,
    .spectrum-FieldLabel--left {
        padding-right: var(--spectrum-global-dimension-size-200);
    }
</style>
