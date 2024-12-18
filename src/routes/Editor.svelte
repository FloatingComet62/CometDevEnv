<script lang="ts">
  import { onMount } from "svelte";
  let {
    file_data,
    save_data,
    cursor: initialCursor,
    update_cursor,
  } = $props<{
    file_data: string;
    save_data: (data: string) => void;
    cursor: { top: number; left: number };
    update_cursor: (cursor: { top: number; left: number }) => void;
  }>();

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

    update_cursor({ top: cursor.top, left: cursor.left });
    cursorDOM.style.left = `${cursor.leftPx()}px`;
    cursorDOM.style.top = `${cursor.topPx()}px`;
  }
  function isTypeable(key: string): boolean {
    return "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890!@#$%^&*()_+-=[]{}\\|;:'\",.<>/?`~ ".includes(
      key
    );
  }

  const TAB_SIZE = 2;

  let INITIAL_LEFT = 0;
  let INITIAL_TOP = 0;
  let STEP_WIDTH = 13.2;
  let STEP_HEIGHT = 28.8;
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
      return INITIAL_LEFT + this.left * STEP_WIDTH;
    }
    topPx() {
      return INITIAL_TOP + this.top * STEP_HEIGHT;
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

      save_data(this.contentChars.join("").replace(/\u00A0/g, " "));
    }

    remove(index: number): number[] {
      this.contentChars.splice(index, 1);
      this.contentChars = [...this.contentChars];

      let old_limits = [...this.limits];
      this.limits = calculateLimits(this.contentChars);

      save_data(this.contentChars.join("").replace(/\u00A0/g, " "));
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

  let cursorDOM: HTMLDivElement;
  let mainElement: HTMLElement;
  let characterElement: HTMLSpanElement;

  onMount(() => {
    const rect = mainElement.getBoundingClientRect();
    INITIAL_LEFT = rect.left;
    INITIAL_TOP = rect.top;

    const characterRect = characterElement.getBoundingClientRect();
    STEP_WIDTH = characterRect.width;
    STEP_HEIGHT = characterRect.height;
    characterElement.style.display = "none";

    updateCursor(cursor, cursorDOM, content, "init");
    //     console.log(
    //       `Spacing configuration:
    // Top Left: { left: ${INITIAL_LEFT}, top: ${INITIAL_TOP} }
    // Step: { width: ${STEP_WIDTH}, height: ${STEP_HEIGHT} }`
    //     );
  });

  let cursor = new Cursor(initialCursor.left, initialCursor.top);
  let content = new Content(file_data);

  // handle movement
  window.addEventListener("keydown", ({ key, ctrlKey }) => {
    if (cursorDOM === null || !movementKeys.includes(key)) return;

    if (key === "ArrowLeft") {
      cursor.deactivateScrolling();
      do {
        cursor.left--;
      } while (
        ctrlKey &&
        cursor.left > 0 &&
        content.contentChars[calculateIndex(content, cursor) - 1] !== "\u00A0"
      );
    } else if (key === "ArrowRight") {
      cursor.deactivateScrolling();
      do {
        cursor.left++;
      } while (
        ctrlKey &&
        cursor.left < content.getLineLimit(cursor) &&
        content.contentChars[calculateIndex(content, cursor) - 1] !== "\u00A0"
      );
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

    let { key } = e;
    if (cursorDOM === null || movementKeys.includes(key)) return;
    let index = calculateIndex(content, cursor);

    if (key === "Backspace") {
      if (index === 0) return;
      function removeChar(i: number) {
        let char = content.contentChars[i];
        let old_limits = content.remove(i);

        if (char == "\n") {
          cursor.top--;
          cursor.left = old_limits[cursor.top] + cursor.left;
        } else {
          cursor.left--;
        }
      }

      if (e.ctrlKey) {
        do {
          removeChar(index - 1);
          index--;
          if (index <= 0) break;
        } while (content.contentChars[index - 1] !== "\u00A0");
      } else {
        removeChar(index - 1);
      }
    } else if (key == "Enter") {
      content.insert(index, ["\n"]);
      cursor.top++;
      cursor.left = 0;
    } else if (key == "Tab") {
      content.insert(index, Array(TAB_SIZE).fill(" "));
      cursor.left += TAB_SIZE;
    } else if (isTypeable(key)) {
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

<main bind:this={mainElement} class="container">
  <div class="text">
    <span bind:this={characterElement}>#</span>
    <div class="cursor" bind:this={cursorDOM}></div>
    {#each content.contentChars as messageChar}
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
    height: 80vh;
    width: 80vw;
    cursor: text;
    font-family: monospace;
    color: var(--text);
    font-size: 1.5rem;
  }
</style>
