<script lang="ts">
  function calculateLimits(messageChars: string[]): number[] {
    let left = 0;
    let output = [];
    for (let i = 0; i < messageChars.length; i++) {
      if (messageChars[i] !== "\n") {
        left++;
        continue;
      }
      output.push(left);
      left = 0;
    }
    output.push(left);
    return output;
  }
  function calculateIndex(limits: number[], left: number, top: number): number {
    let index = 0;
    for (let i = 0; i < top; i++) {
      index += limits[i] + 1;
    }
    index += left;
    return index;
  }

  let cursor: HTMLDivElement | null;

  let message = `console.log("Hello, World!");\nbruh my anus`;
  let messageChars = message.split("");
  let limits = calculateLimits(messageChars);

  let initialLeft = 0;
  let initialTop = 0;
  let leftIndex = 0;
  let topIndex = 0;
  let scrollLeftIndex = 0;
  const STEP_WIDTH = 13.2;
  const STEP_HEIGHT = 28.8;
  function left(n: number): number {
    return initialLeft + n * STEP_WIDTH;
  }
  function top(n: number): number {
    return initialTop + n * STEP_HEIGHT;
  }

  function updateCursor(checkScroll: boolean) {
    if (topIndex < 0) {
      topIndex = 0;
    } else if (topIndex >= limits.length) {
      topIndex = limits.length - 1;
    }

    if (leftIndex < 0) {
      if (topIndex === 0) {
        leftIndex = 0;
      } else {
        topIndex--;
        leftIndex = limits[topIndex];
      }
    } else if (leftIndex > limits[topIndex]) {
      leftIndex = limits[topIndex];
      if (topIndex < limits.length - 1) {
        topIndex++;
        leftIndex = 0;
      }
    } else if (checkScroll && leftIndex != scrollLeftIndex) {
      leftIndex = scrollLeftIndex;
    }

    cursor!.style.left = `${left(leftIndex)}px`;
    cursor!.style.top = `${top(topIndex)}px`;
  }

  function isTypeable(key: string): boolean {
    return "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890!@#$%^&*()_+-=[]{}\\|;:'\",.<>/?`~ ".includes(
      key
    );
  }

  // handle movement
  const movementKeys = [
    "ArrowLeft",
    "ArrowRight",
    "ArrowUp",
    "ArrowDown",
    "Home",
    "End",
  ];
  window.addEventListener("keydown", ({ key }) => {
    if (cursor === null || !movementKeys.includes(key)) return;
    console.log(limits, leftIndex, topIndex);

    let checkScroll = false;
    if (key === "ArrowLeft") {
      leftIndex--;
    } else if (key === "ArrowRight") {
      leftIndex++;
    } else if (key === "ArrowUp") {
      checkScroll = true;
      if (scrollLeftIndex === -1) {
        scrollLeftIndex = leftIndex;
      }
      topIndex--;
    } else if (key === "ArrowDown") {
      checkScroll = true;
      if (scrollLeftIndex === -1) {
        scrollLeftIndex = leftIndex;
      }
      topIndex++;
    } else if (key === "Home") {
      leftIndex = 0;
    } else if (key === "End") {
      leftIndex = limits[topIndex];
    }

    if (!checkScroll) {
      scrollLeftIndex = -1;
    }

    updateCursor(checkScroll);
  });

  // handle keyboard inputs
  window.addEventListener("keydown", ({ key }) => {
    if (cursor === null || movementKeys.includes(key)) return;
    let index = calculateIndex(limits, leftIndex, topIndex);

    if (key === "Backspace") {
      let char = messageChars.splice(index - 1, 1)[0];
      messageChars = [...messageChars];

      if (char == "\n") {
        topIndex--;
        leftIndex = limits[topIndex] + leftIndex;
      } else {
        leftIndex--;
      }

      limits = calculateLimits(messageChars);
    }
    if (isTypeable(key)) {
      messageChars.splice(index, 0, key);
      messageChars = [...messageChars];

      leftIndex++;

      limits = calculateLimits(messageChars);
    }

    updateCursor(false);
  });
</script>

<main class="container">
  <div class="text">
    <div class="cursor" bind:this={cursor}></div>
    {#each messageChars as messageChar}
      {#if messageChar === "\n"}
        <br />
      {:else}
        <span>{messageChar}</span>
      {/if}
    {/each}
  </div>
</main>

<style>
  .cursor {
    position: absolute;
    width: 0.1rem;
    height: 1.5rem;
    background-color: black;
    animation: blink 1s infinite;
  }

  @keyframes blink {
    0% {
      opacity: 1;
    }
    50% {
      opacity: 0;
    }
    100% {
      opacity: 1;
    }
  }

  .container {
    display: flex;
    height: 90vh;
    width: 90vw;
    cursor: text;
    font-family: monospace;
  }
  .text {
    font-size: 1.5rem;
  }
</style>
