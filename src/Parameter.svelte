<script>
import { createEventDispatcher } from "svelte";

export let type = "text";
export let label = "";
export let prepend = "";
export let append = "";
export let value = "";
export let disabled = false;
export let placeholder = "";
export let help = "";

// Execute command when press Enter
let dispatch = createEventDispatcher();
function handleKeydown(event) {
	if(event.key == "Enter")
		dispatch("launch");
}

$: id = `param-${type}-${label}`;
</script>


<div class="form-group row mb-0">
    <label class="col-6 col-form-label" for={id}>{label}</label>

    <div class="col-4 p-0">
        {#if type == "text"}
            <div class="input-group input-group-sm">
                {#if prepend != ""}
                <div class="input-group-prepend">
                    <div class="input-group-text">{prepend}</div>
                </div>
                {/if}

                <input id={id} type="text" on:keydown={handleKeydown} class="form-control form-control-sm" bind:value={value} placeholder={placeholder} disabled={disabled}>

                {#if append != ""}
                <div class="input-group-append">
                    <div class="input-group-text">{append}</div>
                </div>
                {/if}
            </div>

        {:else if type == "checkbox"}
            <div class="custom-control custom-control custom-checkbox mt-2">
                <input id={id} type="checkbox" bind:checked={value} class="custom-control-input" disabled={disabled}>
                <label class="custom-control-label" for={id}></label>
            </div>
        {/if}
    </div>

    <div class="col-1 p-0 pl-3">
        {#if help != ""}
        <button
            type="button"
            class="btn btn-sm btn-outline-primary"
            data-toggle="popover"
            data-trigger="hover"
            data-html="true"
            data-content="{help}">?</button>
        {/if}
    </div>
</div>
