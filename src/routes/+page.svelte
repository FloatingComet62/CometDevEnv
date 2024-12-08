<script lang="ts">
  function min(a: number, b: number): number {
    return a < b ? a : b;
  }
  function max(a: number, b: number): number {
    return a > b ? a : b;
  }
  function clamp(value: number, minValue: number, maxValue: number): number {
    return min(max(value, minValue), maxValue);
  }

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
  function calculateIndex(content: Content, cursor: Cursor): number {
    let index = 0;
    for (let i = 0; i < cursor.top; i++) {
      index += content.limits[i] + 1;
    }
    index += cursor.left;
    return index;
  }
  function updateCursor(
    cursor: Cursor,
    cursorDOM: HTMLDivElement,
    content: Content,
    event: string
  ) {
    cursor.top = clamp(cursor.top, 0, content.limits.length - 1);

    if (cursor.left < 0 && cursor.top === 0) {
      cursor.left = 0;
    } else if (cursor.left < 0) {
      cursor.top--;
      cursor.left = content.getLineLimit(cursor);
    } else if (cursor.left > content.getLineLimit(cursor)) {
      cursor.left = content.getLineLimit(cursor);
      // Explanation for event == "ArrowLeft"
      // The if statement below is designed for the cursor to wrap around to
      // the next line if the user pressed left on the end of the line.
      // But if the user is moving up and down the lines, we encounter
      // a similar state where left > limit and there are lines below which we
      // can wrap to.
      if (cursor.top < content.limits.length - 1 && event == "ArrowLeft") {
        cursor.top++;
        cursor.left = 0;
      }
    } else if (cursor.scrollingActive && cursor.left != cursor.scrollingLeft) {
      cursor.left = min(cursor.scrollingLeft, content.getLineLimit(cursor));
    }

    cursorDOM.style.left = `${cursor.leftPx()}px`;
    cursorDOM.style.top = `${cursor.topPx()}px`;
  }
  function isTypeable(key: string): boolean {
    return "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890!@#$%^&*()_+-=[]{}\\|;:'\",.<>/?`~ ".includes(
      key
    );
  }

  const STEP_WIDTH = 13.2;
  const STEP_HEIGHT = 28.8;
  const TAB_SIZE = 2;
  class Cursor {
    left = $state(0);
    top = $state(0);
    scrollingLeft = $state(-1);
    scrollingActive = $state(false);

    constructor(left: number, top: number) {
      this.left = left;
      this.top = top;
    }

    actiavteScrolling() {
      if (this.scrollingActive) return;
      this.scrollingActive = true;
      this.scrollingLeft = this.left;
    }

    deactivateScrolling() {
      this.scrollingActive = false;
    }

    leftPx() {
      return this.left * STEP_WIDTH;
    }
    topPx() {
      return this.top * STEP_HEIGHT;
    }
  }

  class Content {
    contentChars: string[] = $state([]);
    limits: number[] = $state([]);

    constructor(content: string) {
      this.contentChars = content.replace(/ /g, "\u00A0").split("");
      this.limits = calculateLimits(this.contentChars);
    }

    insert(index: number, chars: string[]) {
      for (let i = chars.length - 1; i >= 0; i--) {
        let char = chars[i];
        if (char == " ") char = "\u00A0";
        this.contentChars.splice(index, 0, char);
      }
      this.contentChars = [...this.contentChars];
      this.limits = calculateLimits(this.contentChars);
    }

    remove(index: number): number[] {
      this.contentChars.splice(index, 1);
      this.contentChars = [...this.contentChars];

      let old_limits = [...this.limits];
      this.limits = calculateLimits(this.contentChars);
      return old_limits;
    }

    getLineLimit(cursor: Cursor) {
      return this.limits[cursor.top];
    }
  }

  const movementKeys = [
    "ArrowLeft",
    "ArrowRight",
    "ArrowUp",
    "ArrowDown",
    "Home",
    "End",
  ];

  let cursorDOM: HTMLDivElement | null;

  let file_data = `#include <stdio.h>\n\nint main() {\n    printf("Hello, World!");\n    return 0;\n}`;

  let cursor = new Cursor(0, 0);
  let content = new Content(file_data);

  // handle movement
  window.addEventListener("keydown", ({ key, preventDefault }) => {
    if (cursorDOM === null || !movementKeys.includes(key)) return;

    if (key === "ArrowLeft") {
      cursor.deactivateScrolling();
      cursor.left--;
    } else if (key === "ArrowRight") {
      cursor.deactivateScrolling();
      cursor.left++;
    } else if (key === "ArrowUp") {
      cursor.actiavteScrolling();
      cursor.top--;
    } else if (key === "ArrowDown") {
      cursor.actiavteScrolling();
      cursor.top++;
    } else if (key === "Home") {
      cursor.deactivateScrolling();
      cursor.left = 0;
    } else if (key === "End") {
      cursor.deactivateScrolling();
      cursor.left = content.limits[cursor.top];
    }

    updateCursor(cursor, cursorDOM, content, key);
  });

  // handle typing inputs
  window.addEventListener("keydown", (e) => {
    // https://stackoverflow.com/questions/49616305/destructing-ev-preventdefault
    e.preventDefault();

    let { key, ctrlKey, altKey, metaKey } = e;
    if (
      cursorDOM === null ||
      movementKeys.includes(key) ||
      ctrlKey ||
      altKey ||
      metaKey
    )
      return;
    let index = calculateIndex(content, cursor);

    if (key === "Backspace") {
      if (index === 0) return;
      let char = content.contentChars[index - 1];
      let old_limits = content.remove(index - 1);

      if (char == "\n") {
        cursor.top--;
        cursor.left = old_limits[cursor.top] + cursor.left;
      } else {
        cursor.left--;
      }
    }
    if (key == "Enter") {
      content.insert(index, ["\n"]);
      cursor.top++;
      cursor.left = 0;
    }
    if (key == "Tab") {
      content.insert(index, Array(TAB_SIZE).fill(" "));
      cursor.left += TAB_SIZE;
    }
    if (isTypeable(key)) {
      content.insert(index, [key]);
      cursor.left++;
    }

    cursor.scrollingActive = false;
    updateCursor(cursor, cursorDOM, content, "typing");
  });

  const keyboardShortCuts = {
    Ctrlv: () => {
      navigator.clipboard.readText().then((text) => {
        if (cursorDOM === null) return;
        let index = calculateIndex(content, cursor);
        content.insert(index, text.split(""));
        cursor.left += text.length;
        cursor.scrollingActive = false;
        updateCursor(cursor, cursorDOM, content, "typing");
      });
    },
  };

  // handle keyboard shortcuts
  window.addEventListener("keydown", (e) => {
    e.preventDefault();
    const { key, ctrlKey, altKey, metaKey } = e;
    const noSpecial = !ctrlKey && !altKey && !metaKey;
    if (cursor === null || noSpecial) return;
    const combination = [];
    if (ctrlKey) combination.push("Ctrl");
    if (altKey) combination.push("Alt");
    if (metaKey) combination.push("Meta");
    combination.push(key);

    for (let [shortcut, action] of Object.entries(keyboardShortCuts)) {
      if (shortcut === combination.join("")) {
        action();
      }
    }
  });
</script>

<main class="container">
  <div class="text">
    <div class="cursor" bind:this={cursorDOM}></div>
    {#each content.contentChars as messageChar, i}
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
    background-color: var(--cursor);
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
    color: var(--text);
  }
  .text {
    font-size: 1.5rem;
  }
</style>
