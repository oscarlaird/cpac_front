<script>
  let state = "AZ";
  let year = "2024";
  let chamber = "senate";
  let goodRollCalls = `https://legiscan.com/AZ/rollcall/SB1003/id/1373207
https://legiscan.com/AZ/rollcall/SB1004/id/1398904`;
  let badRollCalls = "";
  let message = "Enter links to roll calls from the legiscan website.";

  let ws;
  let connected = false;
  import { page } from '$app/stores';
  $: url = $page.url.searchParams.get('serve_local') ? "ws://localhost:5000/ws"
    : "wss://lairdscanback.oscarlaird.com/ws";

  function connectWebSocket() {
    ws = new WebSocket(url);
    ws.onopen = () => {
      connected = true;
      message = "Connected to server";
    };
    
    ws.onmessage = (event) => {
      try {
        const data = JSON.parse(event.data);
        
        if (data.type === "file") {
          // Convert base64 to blob and download
          const bytes = atob(data.data);
          const buffer = new ArrayBuffer(bytes.length);
          const view = new Uint8Array(buffer);
          for (let i = 0; i < bytes.length; i++) {
            view[i] = bytes.charCodeAt(i);
          }
          const blob = new Blob([buffer], { 
            type: 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet' 
          });
          
          const url = window.URL.createObjectURL(blob);
          const a = document.createElement('a');
          a.href = url;
          a.download = data.filename;
          document.body.appendChild(a);
          a.click();
          window.URL.revokeObjectURL(url);
          document.body.removeChild(a);
          
          message = "Download complete!";
        } else if (data.type === "error") {
          message = `Error: ${data.message}`;
        } else {
          // Regular status message
          message = event.data;
        }
      } catch (e) {
        // If it's not JSON, treat it as a regular message
        message = event.data;
      }
    };
    
    ws.onclose = () => {
      connected = false;
      message = "Disconnected from server";
      setTimeout(connectWebSocket, 1000);
    };
    
    ws.onerror = (error) => {
      console.error('WebSocket error:', error);
      message = "WebSocket error occurred";
    };
  }

  // Connect when component mounts
  import { onMount } from 'svelte';
  onMount(() => {
    connectWebSocket();
  });

  async function handleSubmit() {
    // validate inputs
    if (!state || !year || !chamber || !goodRollCalls) {
      message = "Please fill out all fields.";
      return;
    }
    if (state.length !== 2) {
      message = "Please enter a valid state.";
      return;
    }
    if (year.length !== 4) {
      message = "Please enter a valid year.";
      return;
    }
    if (chamber !== "senate" && chamber !== "house") {
      message = "Please enter a valid chamber.";
      return;
    }
    // TODO: Implement form submission
    message = "Processing...";
    try {
      ws.send(JSON.stringify({
        state,
        year,
        chamber,
        goodRollCalls,
        badRollCalls
      }));
    } catch (error) {
      console.error("Error:", error);
      message = `Error: ${error.message}`;
    }
  }
</script>

<div class="container mx-auto p-4 max-w-4xl">
  <div class="bg-white shadow-md rounded px-8 pt-6 pb-8 mb-4">
    
    <div class="mb-4 text-center">
      <p class="text-gray-700">{message}</p>
    </div>

    <div class="flex gap-4 mb-4">
      <div class="flex-1">
        <label class="block text-gray-700 text-sm font-bold mb-2" for="state">
          STATE
        </label>
        <input 
          bind:value={state}
          class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
          id="state"
          type="text"
        />
      </div>

      <div class="flex-1">
        <label class="block text-gray-700 text-sm font-bold mb-2" for="year">
          YEAR
        </label>
        <input 
          bind:value={year}
          class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
          id="year"
          type="text"
        />
      </div>
    </div>

    <div class="mb-4">
      <div class="flex justify-center gap-2">
        <button 
          class={`px-4 py-2 rounded ${chamber === 'senate' ? 'bg-blue-500 text-white' : 'bg-gray-200'}`}
          on:click={() => chamber = 'senate'}
        >
          Senate
        </button>
        <button
          class={`px-4 py-2 rounded ${chamber === 'house' ? 'bg-blue-500 text-white' : 'bg-gray-200'}`}
          on:click={() => chamber = 'house'}
        >
          House
        </button>
      </div>
    </div>

    <div class="mb-4">
      <label class="block text-gray-700 text-sm font-bold mb-2">
        Good Roll Call URLs (one per line)
      </label>
      <textarea
        bind:value={goodRollCalls}
        class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
        rows="5"
      ></textarea>
    </div>

    <div class="mb-6">
      <label class="block text-gray-700 text-sm font-bold mb-2">
        Bad Roll Call URLs (one per line)
      </label>
      <textarea
        bind:value={badRollCalls}
        class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
        rows="5"
      ></textarea>
    </div>

    <div class="flex items-center justify-center">
      <button
        class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline"
        on:click={handleSubmit}
      >
        Create Spreadsheet!
      </button>
    </div>

  </div>
</div>
