<script>
  import { listen } from "@tauri-apps/api/event";
  import VirtualList from "../components/utils/VirtualList.svelte";
  import LogMessage from "../components/log/LogMessage.svelte";
  import { onMount } from "svelte";
  import { minecraftLogs } from "../stores/logsStore.js";
  import { invoke } from "@tauri-apps/api";
  import { addNotification } from "../stores/notificationStore.js";

  let autoScroll = true;

  let searchQuery = "";
  let filteredLogs = [];
  let logLevel;
  let logCounts = {};

  // Track filter state for each log level
  let logLevels = {
    debug: false,
    info: false,
    warn: false,
    error: false,
    fatal: false,
  };

  onMount(() => {
  invoke("get_latest_minecraft_logs").then(value => {
    minecraftLogs.set(value.map(string => string + "\n"));
  }).catch(reason => {
    addNotification(reason);
  });

  let logsUnlisten;

  listen("process-output", event => {
    minecraftLogs.update(value => [...value, event.payload]);
  }).then(unlisten => {
    logsUnlisten = unlisten;
  });

  return () => {
    if (logsUnlisten) logsUnlisten();
  };
});

  async function uploadLogs() {
    await invoke("upload_logs", {
      log: $minecraftLogs.join(""),
    }).then((result) => {
      addNotification("Logs uploaded successfully. URL copied to clipboard.", "INFO");
      navigator.clipboard.writeText(result.url);
    }).catch((error) => {
      addNotification(error);
    });
  }

  function toggleAutoScroll() {
    autoScroll = !autoScroll;
  }

  $: {
    minecraftLogs.subscribe(logs => {
      const isAnyLevelSelected = Object.values(logLevels).some(level => level);
      filteredLogs = logs.filter(log => {
        const logMatch = log.match(/\[(.*?)\]/g);
        if (logMatch && logMatch.length > 1) {
          logLevel = logMatch[1].split('/')[1].replace(']', '');
        }
        if (isAnyLevelSelected) {
          return logLevels[logLevel.toLowerCase()] && log.toLowerCase().includes(searchQuery.toLowerCase());
        } else {
          return log.toLowerCase().includes(searchQuery.toLowerCase());
        }
      });
    });
  
    if (searchQuery !== "") {
      autoScroll = false;
    } else {
      autoScroll = true;
    }
    
    Object.keys(logLevels).forEach(level => {
      logCounts[level] = $minecraftLogs.filter(log => {
        const logMatch = log.match(/\[(.*?)\]/g);
        if (logMatch && logMatch.length > 1) {
          const logLevel = logMatch[1].split('/')[1].replace(']', '');
          return logLevel.toLowerCase() === level;
        }
        return false;
      }).length;
    });
  }

  function highlightSearchQuery(text) {
    if (searchQuery === "") {
      return text;
    }
    const parts = text.split(new RegExp(`(${searchQuery})`, 'gi'));
    return parts.map(part => part.toLowerCase() === searchQuery.toLowerCase() ? `<span style="background-color:yellow;">${part}</span>` : part).join('');
  }
</script>

<div class="black-bar">
<input class="search-bar" type="text" bind:value={searchQuery} placeholder="Search logs...">
<div class="filter-options">
  {#each Object.keys(logLevels) as level (level)}
    <label>
      <input
        type="checkbox"
        class="ui-checkbox"
        class:checked={Array.from(Object.values(logLevels)).every(level => level === false)}
        checked={logLevels[level]}
        on:change={() => {
          logLevels[level] = !logLevels[level];
        }}
      />
      <!-- First letter uppercase -->
      {level.at(0).toUpperCase() + level.slice(1)} ({logCounts[level]})
    </label>
  {/each}
</div>
</div>
<main class="content">
  <div class="logs-wrapper">
    <VirtualList items={filteredLogs} let:item {autoScroll}>
      <LogMessage text={highlightSearchQuery(item)} />
    </VirtualList>
  </div>
</main>

<div class="black-bar" data-tauri-drag-region>
  <div class="logs-button-wrapper">
    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <h1
      class:auto-scroll-button-on={autoScroll}
      class:green-text={autoScroll}
      class:auto-scroll-button-off={!autoScroll}
      class:red-text={!autoScroll}
      on:click={toggleAutoScroll}
    >[Auto Scroll]</h1>
    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <h1 class="copy-button primary-text" on:click={uploadLogs}>
      [Copy]
    </h1>
  </div>
</div>

<style>
  .search-bar {
    margin-right: 20px;
    padding: 10px;
    border: 3px solid #ccc;
    border-radius: 0; 
    outline: none;
    box-shadow: 0 0 10px rgba(0,0,0,0.2);
    font-family: 'Press Start 2P', serif;
    font-size: 10px;
  }

  .black-bar {
    display: flex;
    align-content: center;
    justify-content: center;
    align-items: center;
    width: 100%;
    height: 10vh;
    background-color: #151515;
  }

  .content {
    height: 80vh;
  }

  .logs-wrapper {
    height: 100%;
    display: flex;
    flex-direction: column;
    gap: 1em;
  }

  .logs-button-wrapper {
    display: flex;
    justify-content: space-between;
    padding: 1em;
    gap: 2em;
  }

  .filter-options {
    display: flex;
    gap: 1em;
  }

  .black-bar label{
    font-family: 'Press Start 2P', serif;
  }

  .black-bar label {
    cursor: pointer;
    color: #ccc;
    text-shadow: 2px 2px #7a7777;
    font-size: 8px;
    display: flex;
    gap: 0.5em;
    align-items: center;
  }

  .copy-button {
    transition: 0.3s;
    font-family: 'Press Start 2P', serif;
    font-size: 17px;
    cursor: pointer;
  }

  .auto-scroll-button-on {
    transition: 0.3s;
    font-family: 'Press Start 2P', serif;
    font-size: 17px;
    cursor: pointer;
  }

  .auto-scroll-button-off {
    transition: 0.3s;
    font-family: 'Press Start 2P', serif;
    font-size: 17px;
    cursor: pointer;
  }

  .copy-button:hover,
  .auto-scroll-button-on:hover,
  .auto-scroll-button-off:hover {
    transform: scale(1.2);
  }

  .ui-checkbox {
  --primary-color: var(--green-text);
  --secondary-color: #151515;
  --primary-hover-color: var(--primary-text);
  /* checkbox */
  --checkbox-diameter: 14px;
  --checkbox-border-radius: 0px;
  --checkbox-border-color: #ccc;
  --checkbox-border-width: 1px;
  --checkbox-border-style: solid;
  /* checkmark */
  --checkmark-size: 1;
}

.ui-checkbox {
  -webkit-appearance: none;
  -moz-appearance: none;
  appearance: none;
  width: var(--checkbox-diameter);
  height: var(--checkbox-diameter);
  border-radius: var(--checkbox-border-radius);
  background: var(--secondary-color);
  border: var(--checkbox-border-width) var(--checkbox-border-style) var(--checkbox-border-color);
  cursor: pointer;
  position: relative;
}

.ui-checkbox::after {
  content: "";
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  -webkit-box-shadow: 0 0 0 calc(var(--checkbox-diameter) / 2.5) var(--primary-color);
  box-shadow: 0 0 0 calc(var(--checkbox-diameter) / 2.5) var(--primary-color);
  border-radius: inherit;
  opacity: 0;
}

.ui-checkbox::before {
  top: 40%;
  left: 50%;
  content: "";
  position: absolute;
  width: 4px;
  height: 7px;
  border-right: 2px solid var(--secondary-color);
  border-bottom: 2px solid var(--secondary-color);
  -webkit-transform: translate(-50%, -50%) rotate(45deg) scale(0);
  -ms-transform: translate(-50%, -50%) rotate(45deg) scale(0);
  transform: translate(-50%, -50%) rotate(45deg) scale(0);
  opacity: 0;
}

/* actions */

.ui-checkbox:hover {
  border-color: var(--primary-color);
}

.ui-checkbox:checked,
.ui-checkbox.checked {
  background: var(--primary-color);
  border-color: transparent;
}

.ui-checkbox:checked::before,
.ui-checkbox.checked::before {
  opacity: 1;
  -webkit-transform: translate(-50%, -50%) rotate(45deg) scale(var(--checkmark-size));
  -ms-transform: translate(-50%, -50%) rotate(45deg) scale(var(--checkmark-size));
  transform: translate(-50%, -50%) rotate(45deg) scale(var(--checkmark-size));
}

.ui-checkbox:active:not(:checked)::after {
  -webkit-transition: none;
  -o-transition: none;
  -webkit-box-shadow: none;
  box-shadow: none;
  transition: none;
  opacity: 1;
}
</style>
