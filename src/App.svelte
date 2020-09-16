<!--
  (C) 2020 David Lettier
  lettier.com
-->

<script>
  import { onMount } from 'svelte';

  let show = false;
  let totalVisibleKeys = 0;
  let currentKey = null;
  let currentKeyIndex = 0;
  let keymap = [];
  let flip = false;
  let dataSaved = false;
  let reverse = false;
  let filtering = false;
  let input;

  function updateTotalVisibleKeys() {
    totalVisibleKeys = keymap.reduce((a, b) => {
      if (b.hidden) { return a; }
      return a + 1;
    }, 0);

    if (totalVisibleKeys <= 0) {
      filtering = true;
    }
  }

  function updateProgress() {
    updateTotalVisibleKeys();

    let count = 0;

    currentKeyIndex = 0;

    for (let i = 0; i < keymap.length; ++i) {
      let key = keymap[i];

      if (key.hidden) { continue; }
      if (key.id === currentKey.id) { currentKeyIndex = count; break; }

      count += 1;
    }
  }

  onMount(async () => {
    let logo = document.getElementById('logo');
    let animationEnd = () => { show = true; };

    logo.addEventListener("animationend",       animationEnd);
    logo.addEventListener("webkitAnimationend", animationEnd);

    let cache = window.localStorage.getItem('blender-keymap-trainer');

    try {
      cache = JSON.parse(cache);
    } catch {
      cache = null;
    }

    if (cache && cache.currentKey && cache.keymap) {
      keymap = cache.keymap;
      currentKey = cache.currentKey;
      currentKeyIndex = cache.currentKeyIndex;
      reverse = cache.reverse === true;

      if (currentKey.hidden) { nextKey(0, false); }

      dataSaved = true;
    } else {
      const response = await fetch('keymap.json');
      let keymapRaw = await response.json();

      processRawKeymap(keymapRaw);
    }

    updateProgress();
  });

  function processRawKeymap(keymapRaw) {
    let keymapTemp = [];

    keymapRaw.forEach((e) => {
      let section = {};
      let items   = [];

      section.title  = e[0];
      section.space  = e[1].space_type.replace(/_/g, " ");
      section.region = e[1].region_type.replace(/_/g, " ");

      e[2].items.forEach((i) => {
        if (i[1] &&
            i[1].value &&
            (i[1].value.toLowerCase() === "press" ||
             i[1].value.toLowerCase() === "click")
        ) {
          let item = {};

          let action = i[0];
          action = action.split(".");
          if (action[1]) {
            action[0] = `(${action[0]})`;
            action[1] = action[1].replace(/_/g, " ");
            item.action = `${action[0]} ${action[1]}`;
          } else {
            item.action = action[0].replace(/_/g, " ");
          }

          item.key   = i[1].type.replace(/_/g, " ");
          item.ctrl  = i[1].ctrl ? true : false;
          item.shift = i[1].shift ? true : false;

          item.properties = [];
          if (i[2] && i[2].properties && i[2].properties.length > 0) {
            i[2].properties.forEach((e) => {
              if (e[1] === "") { return; }

              let k = String(e[0]).replace(/_/g, " ");
              let v = String(e[1]).replace(/_/g, " ").replace(/\./g, " ");

              item.properties.push(
                `${k}: ${v}`
              );
            });
          }

          item.section = section;

          item.passed = 0;
          item.failed = 0;

          item.hidden = false;

          item.id = keymapTemp.length;

          items.push(item);

          keymapTemp.push(item);
        }
      });
    });

    keymap = keymapTemp;

    currentKeyIndex = Math.floor(Math.random() * keymap.length);
    currentKey = keymap[currentKeyIndex];
  };

  function nextKey(passed, skip) {
    if (passed == 1) {
      currentKey.passed += 1;
    } else if (passed == -1) {
      currentKey.failed += 1;
    }

    updateTotalVisibleKeys();

    if (totalVisibleKeys === 1) {
      for (let i = 0; i < keymap.length; ++i) {
        let key = keymap[i];

        if (!key.hidden) { currentKey = key; break; }
      }
    } else if (totalVisibleKeys > 1 ) {
      if (skip) {
        let index = currentKey.id;

        while (true) {
          index += 1;
          if (index >= keymap.length) { index = 0; }
          let key = keymap[index];
          if (!key.hidden && key.id !== currentKey.id) {
            currentKey = key;
            break;
          }
        }

      } else {
        let visibleKeys = keymap.reduce((a, b) => {
          if (b.hidden || b.id === currentKey.id) { return a; }

          a.push(b);
          return a;
        }, []);
        let totalFailed = visibleKeys.reduce((a, b) => {
          if (b.hidden) { return a; }

          let failed = b.failed - b.passed;
          if (failed < 0) { failed = 0; };

          return a + failed + 1;
        }, 0);

        let number = Math.floor(Math.random() * (totalFailed + 1));
        let count = 0

        for (let i = 0; i < visibleKeys.length; ++i) {
          let key = visibleKeys[i];
          let failed = key.failed + 1;

          count += failed;

          if (number <= count) {
            currentKey = key;
            break;
          }
        }
      }

      currentKey = currentKey;
      keymap = keymap;
    }

    updateProgress();

    flip = false;

    window.localStorage.setItem(
      'blender-keymap-trainer',
      JSON.stringify({
        reverse: reverse,
        currentKey: currentKey,
        currentKeyIndex: currentKeyIndex,
        keymap: keymap
      })
    );

    dataSaved = true;
  }

  function clearData(force) {
    if (force || window.confirm("Clear data?")) {
      window.localStorage.removeItem('blender-keymap-trainer');
      dataSaved = false;
    }
  }

  function upload() {
    let file = input.files[0];
    let fileReader = new FileReader();

    fileReader.readAsText(file);

    fileReader.onload = () => {
      try {
        processRawKeymap(JSON.parse(fileReader.result));
        updateProgress();
        clearData(true);
        window.alert("Keymap uploaded.");
      } catch {
        window.alert("Could not process upload.");
      };
    };
  }

  function updateHidden(type, key) {
    if (type === "key") {
      key.hidden = !key.hidden;
      key = key;
    } else if (type === "section.title") {
      let hidden = !key.hidden;

      keymap.forEach((e) => {
        if (e.section.title !== key.section.title) {
          return;
        }

        e.hidden = hidden;
        e = e;
      });
    } else if (type === "section.region") {
      let hidden = !key.hidden;

      keymap.forEach((e) => {
        if (e.section.region !== key.section.region) {
          return;
        }

        e.hidden = hidden;
        e = e;
      });
    } else if (type === "section.space") {
      let hidden = !key.hidden;

      keymap.forEach((e) => {
        if (e.section.space !== key.section.space) {
          return;
        }

        e.hidden = hidden;
        e = e;
      });
    }

    keymap = keymap;

    nextKey(0, false);
  }
</script>

<main>
    <div class="main-container-column {show ? 'logo-background fade-in': ''}">
      {#if !show}
        <img id="logo" class="logo fade-out" src="logo.svg" alt="logo">
      {/if}
      {#if show && keymap.length > 0 && !filtering}
        <div class="main-container-row fade-in">
          <div></div>
          <div class="card">
            {#if currentKey}
              <div class="count">
                {currentKeyIndex + 1} / {totalVisibleKeys}
              </div>
              <div class="section-title">
                {currentKey.section.title}
              </div>
              <div class="breadcrumbs">
                <div class="breadcrumb">
                  {currentKey.section.region}
                </div>
                {#if currentKey.section.space !== "EMPTY"}
                  <div class="breadcrumb arrow-right">
                  </div>
                  <div class="breadcrumb">
                    {currentKey.section.space}
                  </div>
                {/if}
              </div>
              <div class="card-content-container {flip ? 'flip-card' : ''}" on:click="{() => flip = !flip}">
                {#if (!flip && !reverse) || (flip && reverse)}
                  <div class="card-content card-content-front {reverse ? 'rotated' : ''}">
                    <div class="card-content-text">
                      {#if currentKey.ctrl}
                        Ctrl +
                      {/if}
                      {#if currentKey.shift}
                        Shift +
                      {/if}
                      {currentKey.key}
                    </div>
                  </div>
                {/if}
                {#if flip || reverse}
                  <div class="card-content card-content-back {reverse ? '' : 'rotated'}">
                    <div class="card-content-text">
                      <div>
                        {currentKey.action}
                      </div>
                      {#each currentKey.properties as property}
                        <div class="properties">
                          {property}
                        </div>
                      {/each}
                    </div>
                  </div>
                {/if}
              </div>
              <div class="answers">
                <div class="button positive-answer {flip ? '' : 'disabled'}"
                    on:click="{ () => { if (flip) nextKey(1, false); }}">
                  {#if flip}👏{/if} Got it!
                </div>
                <div class="button skip-answer {flip ? 'disabled' : ''}" on:click="{ () => { if (!flip) nextKey(0, true); } }">
                  {#if !flip}💤{/if} Skip
                </div>
                <div class="button negative-answer {flip ? '' : 'disabled'}"
                    on:click="{ () => { if (flip) nextKey(-1, false); }}">
                  {#if flip}🤕{/if} Missed it!
                </div>
              </div>
            {/if}
            <div class="controls">
              <div class="button reverse-control margin-right {flip ? '' : 'disabled'}"
                   on:click="{ () => { if (flip) reverse = !reverse } }">
                {#if flip}➰{/if} Reverse
              </div>
              <div class="button upload-control margin-right">
                <label class="upload-label">
                  <input type="file" accept="application/json" bind:this={input} on:change="{ () => upload() }">
                  📂 Upload
                </label>
              </div>
              <div class="button filter-control margin-right" on:click="{ () => filtering = true }">
                📝 Filter
              </div>
              <div class="button delete-control {dataSaved ? '' : 'disabled'}"
                   on:click="{ () => { if (dataSaved) clearData(false); } }">
                {#if dataSaved}🗑{/if} Clear
              </div>
            </div>
          </div>
          <div></div>
        </div>
      {/if}
      {#if show && keymap.length && filtering}
        <div class="filtering-container fade-in">
          <div class="count">
            {totalVisibleKeys}
          </div>
          <div class="filter-lines-container">
            {#each keymap as key, i}
              {#if i === 0}
                <div class="filter-line filter-lines-header">
                  <div class="filter-line-item">
                    KEY ACTION
                  </div>
                  <div class="filter-line-item">
                    SECTION
                  </div>
                  <div class="filter-line-item">
                    SPACE
                  </div>
                  <div class="filter-line-item">
                    REGION
                  </div>
                </div>
              {/if}
              <div class="filter-line {key.hidden ? 'disabled' : ''} {(i % 2 === 0) ? 'filter-line-a' : 'filter-line-b'}">
                <div class="filter-line-item" on:click="{ () => updateHidden('key', key) }">
                  {#if key.ctrl}
                    Ctrl +
                  {/if}
                  {#if key.shift}
                    Shift +
                  {/if}
                  {key.key}
                  <br>
                  {key.action}
                </div>
                <div class="filter-line-item" on:click="{ () => updateHidden('section.title', key) }">
                  {key.section.title}
                </div>
                <div class="filter-line-item" on:click="{ () => updateHidden('section.space', key) }">
                  {key.section.space}
                </div>
                <div class="filter-line-item" on:click="{ () => updateHidden('section.region', key) }">
                  {key.section.region}
                </div>
              </div>
            {/each}
          </div>
          <div class="filtering-controls">
            <div class="button done-control {totalVisibleKeys <= 0 ? 'disabled' : ''}"
                 on:click="{ () => { if (totalVisibleKeys > 0) { filtering = false; nextKey(0, false); } } }">
              {#if totalVisibleKeys > 0}📝{/if} Done
            </div>
          </div>
        </div>
      {/if}
    </div>
</main>

<style>
  main {
    border: 0px solid transparent;
    margin: 0px;
    width: 100%;
    height: 100%;
  }

  input[type="file"] {
    display: none;
  }

  .main-container-column {
    display: flex;
    flex-direction: column;
    align-items: center;
    align-content: center;
    justify-content: center;
    height: 100%;
  }

  .logo {
    width: 90%;
  }

  .main-container-row {
    width: 100%;
    height: 60%;
    display: flex;
    flex-direction: row;
    justify-content: space-between;
  }

  .card {
    width: 50%;
    display: flex;
    flex-direction: column;
    perspective: 1000px;
  }

  .card-content-container {
    display: flex;
    flex-direction: row;
    justify-content: center;
    align-items: center;
    height: 20vh;
    cursor: pointer;
    position: relative;
    transition: transform 0.5s;
    transform-style: preserve-3d;
    margin-top: 5px;
    margin-bottom: 5px;
    height: 100%;
  }

  .card-content {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    align-self: center;
    font-size: 6vh;
    position: absolute;
    height: 40%;
    width: 100%;
    text-align: center;
    width: 100%;
    height: 100%;
    overflow-y: auto;
  }

  .card-content-text {
    padding: 1px;
  }

  .card-content-front {
    background-color: #c7dcd0;
  }

  .card-content-back {
    color: white;
    background-color: #7f708a;
  }

  .flip-card {
    transform: rotateY(180deg);
  }

  .rotated {
    transform: rotateY(180deg);
    z-index: 100;
  }

  .count {
    color: white;
    text-align: center;
    margin-bottom: 5px;
    color: #625565;
  }

  .section-title {
    display: flex;
    flex-direction: row;
    flex-wrap: nowrap;
    justify-content: center;
    align-items: center;
    background-color: #4d9be6;
    padding: 10px;
    color: white;
    font-size: 3.2vh;
    margin-bottom: 5px;
  }

  .breadcrumbs {
    display: flex;
    flex-direction: row;
    flex-wrap: nowrap;
    justify-content: center;
    align-items: center;
    background-color: #484a77;
    padding: 10px;
  }

  .breadcrumb {
    color: white;
    font-size: 3vh;
    margin-left: 10px;
    margin-right: 10px;
  }

  .arrow-right {
    width: 0;
    height: 0;
    margin-left: 10px;
    margin-right: 10px;
    border-top: 10px solid transparent;
    border-bottom: 10px solid transparent;
    border-left: 10px solid white;
  }

  .answers {
    display: flex;
    flex-direction: row;
    flex-wrap: nowrap;
    justify-content: space-around;
    align-items: center;
    height: 15%;
    min-height: 4vh;
  }

  .button {
    display: flex;
    flex-direction: column;
    justify-content: center;
    text-align: center;
    cursor: pointer;
    color: white;
    font-size: 3vh;
    padding-left: 10px;
    padding-right: 10px;
    height: 100%;
    width: 100%;
    -webkit-user-select: none;
    -moz-user-select: none;
    -ms-user-select: none;
    user-select: none;
  }

  .positive-answer {
    background-color: #0eaf9b;
  }

  .negative-answer {
    background-color: #e83b3b;
  }

  .skip-answer {
    background-color: #f04f78;
    margin-right: 5px;
    margin-left: 5px;
  }

  .controls {
    display: flex;
    flex-direction: row;
    flex-wrap: nowrap;
    justify-content: space-around;
    align-items: center;
    margin-top: 5px;
    height: 15%;
    min-height: 4vh;
  }

  .reverse-control {
    background-color: #fb6b1d;
  }

  .upload-control {
    background-color: #4d65b4;
    cursor: pointer;
  }

  .upload-label {
    cursor: pointer;
  }

  .filter-control {
    background-color: #905ea9;
  }

  .delete-control {
    background-color: #c32454;
  }

  .properties {
    font-size: 2vh;
  }

  .filtering-container {
    display: flex;
    flex-direction: column;
    width: 50%;
    height: 60%;
    overflow-y: auto;
    overflow-x: hidden;
  }

  .filter-lines-container {
    display: flex;
    flex-direction: column;
    overflow-y: auto;
    overflow-x: hidden;
  }

  .filtering-controls {
    margin-top: 5px;
    width: 100%;
  }

  .filter-line {
    display: flex;
    flex-direction: row;
    justify-content: stretch;
    width: 100%;
    color: white;
    padding: 5px;
    margin-bottom: 5px;
  }

  .filter-line-item {
    width: 25%;
    padding-left: 5px;
    cursor: pointer;
  }

  .filter-line-item:hover {
    color: #2e222f;
    background-color: #a884f3;
  }

  .filter-lines-header {
    background-color: #753c54;
  }

  .filter-line-a {
    background-color: #3e3546;
  }

  .filter-line-b {
    background-color: #625565;
  }

  .done-control {
    height: 6vh;
    background-color: #905ea9;
  }

  .disabled {
    cursor: default;
    background-color: #3e3546;
    color: #625565;
  }

  .margin-right {
    margin-right: 5px;
  }

  .fade-in {
    opacity: 1;
    animation: fadeIn ease-in-out 1s;
  }

  .fade-out {
    opacity: 0;
    animation: fadeOut ease-in-out 2s;
  }

  @keyframes fadeIn {
    0% {
      opacity: 0;
    }
    100% {
      opacity: 1;
    }
  }

  @keyframes fadeOut {
    0% {
      opacity: 1;
    }
    60% {
      opacity: 0.9;
    }
    100% {
      opacity: 0;
    }
  }

  @media screen and (max-width: 1280px) {
    .main-container-column {
      height: 100%;
      width: 100%;
    }

    .main-container-row {
      width: 100%;
      height: 80%;
    }

    .card {
      width: 85%;
    }

    .filtering-container {
      width: 90%;
      height: 90%;
    }

    .section-title {
      font-size: 3vw;
    }

    .breadcrumb {
      font-size: 2vw;
    }

    .card-content {
      font-size: 4vw;
    }

    .button {
      font-size: 3vw;
    }
  }

  @media screen and (max-width: 600px) {
    .filter-line {
      font-size: 2.8vw;
    }

    .section-title {
      font-size: 4vw;
    }

    .breadcrumb {
      font-size: 4vw;
    }

    .button {
      font-size: 4.5vw;
    }
  }

  @media screen and (max-width: 470px) {
    .button {
      font-size: 4vw;
    }
  }
</style>
