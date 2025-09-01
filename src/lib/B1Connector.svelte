<script lang="ts">
  import MdIcon from "./basic/MdIcon.svelte";
  import { printerClient, initClient, connectionState } from "../stores";
  import type { ConnectionType } from "../types";
  import { Toasts } from "../utils/toasts";
  import { NiimbotSerialClient } from "@mmote/niimbluelib";

  let connectionType: ConnectionType = "serial";

  // NIIMBOT B1 vendor and product IDs
  const TARGET_VENDOR_ID = 0x3513;
  const TARGET_PRODUCT_ID = 0x0002;

  const connectToB1 = async () => {
    try {
      initClient(connectionType);
      connectionState.set("connecting");

      // Connect using the NiimbotSerialClient
      if ($printerClient instanceof NiimbotSerialClient) {
        // We'll override the connect method to add our filtering
        const originalConnect = $printerClient.connect;
        
        // Override the connect method to filter for B1
        $printerClient.connect = async function() {
          // Call the original connect method which will show the browser's port selection
          const result = await originalConnect.call(this);
          
          // Get port info
          if (this.port) {
            const info = this.port.getInfo();
            
            // Check if it matches our target device
            if (
              info.usbVendorId === TARGET_VENDOR_ID &&
              info.usbProductId === TARGET_PRODUCT_ID
            ) {
              // This is our B1 printer, proceed with the connection
              return result;
            } else {
              // This is not our B1 printer, disconnect and show an error
              await this.disconnect();
              throw new Error(`Selected device is not a NIIMBOT B1 printer. Vendor ID: ${info.usbVendorId}, Product ID: ${info.usbProductId}`);
            }
          }
          
          return result;
        };
        
        // Now call the connect method which will use our overridden version
        await $printerClient.connect();
      } else {
        // Fallback to standard connection
        await $printerClient.connect();
      }
    } catch (error) {
      connectionState.set("disconnected");
      console.error("Error connecting to device:", error);
      Toasts.error(`Failed to connect to device: ${error.message}`);
    }
  };
</script>

<button class="btn text-nowrap {connectionType === 'serial' ? 'btn-light' : 'btn-outline-secondary'}" on:click={connectToB1}>
  <MdIcon icon="usb" />
  Connect to B1
</button>