<script>
  import DOMPurify from 'dompurify';
  export let text = "";

  $: sanitizedText = DOMPurify.sanitize(text, {ALLOWED_TAGS: ['span']});
  
  // Process and highlight text
  $: processedText = sanitizedText.replace(/\[([^\]]*)\]/g, (match) => {
      // Check if it's a time (matches the format [HH:MM:SS])
      if (/^\[\d{2}:\d{2}:\d{2}\]$/.test(match)) {
        return `<span class="time">${match}</span>`;
      }
      else if (match.includes("WARN")) {
        return`<span class="warn">${match}</span>`;
      } else if (match.includes("ERROR")) {
        return `<span class="error">${match}</span>`;
      } else if (match.includes("INFO")) {
        return `<span class="info">${match}</span>`;
      } else if (match.includes("FATAL")) {
        return `<span class="fatal">${match}</span>`;
      }

      return match
    });
</script>

<!-- Render the processed log messages -->
<pre class="message">{@html processedText}</pre>

<style>
    .message {
        font-family: monospace;
        font-size: 11px;
        user-select: text;
        word-break: break-word;
        overflow-wrap: anywhere;
        text-shadow: none !important;
    }

    /* Use :global to apply these styles globally, not scoped to the component */
    :global(.message .warn) {
        color: #ff9100;
        text-shadow: none;
    }

    :global(.message .error) {
        font-weight: bold;
        color: red;
        text-shadow: none;
    }

    :global(.message .fatal) {
        font-weight: bold;
        color: red;
        text-shadow: none;
    }

    :global(.message .info) {
        color: var(--primary-color);
        text-shadow: none;
    }

    :global(.message .time) {
        color: #9b9b9b;
        text-shadow: none;
    }
</style>
