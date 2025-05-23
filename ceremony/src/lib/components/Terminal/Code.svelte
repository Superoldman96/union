<script lang="ts">
import Buttons from "$lib/components/Terminal/Install/Buttons.svelte"
import Print from "$lib/components/Terminal/Print.svelte"
import { getState } from "$lib/state/index.svelte.ts"
import { user } from "$lib/state/session.svelte.ts"
import { callJoinQueue, getAverageTimes } from "$lib/supabase"
import { queryQueueCount } from "$lib/supabase/queries.ts"
import { axiom } from "$lib/utils/axiom.ts"
import { formatWaitTime, sleep } from "$lib/utils/utils.ts"
import { onDestroy, onMount } from "svelte"

const { terminal, contributor } = getState()

let inputCode: string = $state("")
let normalizedCode = $derived(normalizeString(inputCode))
let inputElement: HTMLInputElement
let showConfirm = $state(false)
let showInput = $state(true)

async function handleKeyDown(event: KeyboardEvent) {
  if (event.key === "Enter") {
    event.preventDefault()

    let queue = await queryQueueCount()
    const averages = await getAverageTimes()

    terminal.updateHistory({ text: "", lineBreak: true, duplicate: true })
    terminal.updateHistory({ text: `Entered code: ${inputCode}`, duplicate: true })
    terminal.updateHistory({
      text:
        "Warning: you must have your browser open and terminal running when it is your turn to contribute. You cannot leave the queue, and when it is your turn you have 1 hour to contribute.",
      type: "warning",
      duplicate: true,
    })
    if (queue.count !== null) {
      if (queue.count > 0) {
        terminal.updateHistory({ text: "", lineBreak: true, duplicate: true })
        let message = `There ${queue.count === 1 ? "is" : "are"} ${queue.count} ${
          queue.count === 1 ? "person" : "people"
        } ahead of you in the queue.`

        if (averages.totalMs) {
          const waitTimeMinutes = (averages.totalMs / 1000 / 60) * queue.count
          const formattedWaitTime = formatWaitTime(waitTimeMinutes)
          message += ` Average wait time: ${formattedWaitTime}.`
        }

        terminal.updateHistory({
          text: message,
          type: "warning",
          duplicate: true,
        })
      } else {
        terminal.updateHistory({ text: "", lineBreak: true, duplicate: true })
        terminal.updateHistory({
          text: "The queue is currently empty. You'll be the next to contribute if you enter now.",
          type: "warning",
          duplicate: true,
        })
      }
    }

    showInput = false
    showConfirm = true
  }
}

onMount(() => {
  terminal.setStep(7)
  if (inputElement) {
    inputElement.focus()
  }
})

onDestroy(() => {
  if (inputElement) {
    inputElement.blur()
  }
  terminal.clearHistory()
})

function normalizeString(input: string): string {
  return input.toLowerCase().replace(/[^a-z0-9]/gi, "")
}

async function handleCodeJoin() {
  try {
    terminal.updateHistory({ text: "Checking code...", duplicate: true })
    console.log("code: ", normalizedCode)
    await sleep(1000)
    const codeOk = await callJoinQueue(normalizedCode)
    if (codeOk) {
      contributor.setAllowanceState("hasRedeemed")
      terminal.updateHistory({ text: "Code successfully redeemed" })
    } else {
      terminal.updateHistory({ text: "The code is not valid", duplicate: true })
      terminal.updateHistory({ text: "", lineBreak: true, duplicate: true })
      axiom.ingest("monitor", [{ user: user.session?.user.id, type: "invalid_code" }])
      onCancel()
    }
  } catch (error) {
    console.error("Error redeeming code:", error)
    terminal.updateHistory({ text: "An error occurred while redeeming the code" })
    onCancel()
  }
}

function onCancel() {
  showInput = true
  showConfirm = false
}

function trigger(value: "enter" | "cancel") {
  if (value === "enter") {
    handleCodeJoin()
  } else if (value === "cancel") {
    onCancel()
  }
}
</script>

{#if showInput}
  <div class="flex w-full gap-1">
    <div class="whitespace-nowrap">
      <Print>Enter code:</Print>
    </div>
    <input
      autofocus
      bind:this={inputElement}
      bind:value={inputCode}
      onkeydown={handleKeyDown}
      class="inline-flex bg-transparent w-full text-union-accent-500 outline-none focus:ring-0 focus:border-none"
      style="--tw-ring-color: transparent;"
    />
  </div>
{/if}

{#if showConfirm && !showInput}
  <Buttons
    index={1}
    data={[{ text: "Enter the queue", action: "enter" }, { text: "Cancel", action: "cancel" }]}
    trigger={(value: "enter" | "cancel") => trigger(value)}
  />
{/if}
