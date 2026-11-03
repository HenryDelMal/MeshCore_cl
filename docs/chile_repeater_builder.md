# Chile repeater builder and key provisioner

This page connects the existing repeater firmware build to [MeshChile Keygen](https://mc-keygen.meshchile.cl/). The keygen generates the key in your browser and registers the public key in the padrón. Then use the local USB provisioner below to install that identity after flashing.

The private key stays in your browser. It is not sent to GitHub Actions or included in the firmware artifact. Save the keygen backup in a safe place before applying it; anyone with the private key controls that mesh identity.

## Build firmware

Choose a repeater target and start the [Repeater Firmware workflow](https://github.com/HenryDelMal/MeshCore_cl/actions/workflows/build-repeater-firmwares.yml). The workflow builds supported repeater firmware artifacts. Confirm the frequency and radio settings with the local community before transmitting.

## Flash and provision the registered identity

Flash the selected repeater firmware with the supported flasher for that board. Connect the device to USB, open the repeater CLI below in Chrome or Edge, and paste the expanded private key from the keygen backup. Provisioning sends the key directly over USB to the device; the page suppresses the device's echoed command so the key is not displayed in the terminal.

<form data-role="form" onsubmit="return false" style="display:grid;gap:.75rem;max-width:36rem">
  <label>Expanded private key (128 hexadecimal characters)
    <input data-role="key" type="password" autocomplete="off" spellcheck="false" minlength="128" maxlength="128" pattern="[0-9a-fA-F]{128}" required style="display:block;width:100%;font:monospace;padding:.6rem">
  </label>
  <p><button type="button" data-action="connect">Connect USB</button>
  <button type="button" data-action="apply" disabled>Set key and reboot</button>
  <button type="button" data-action="disconnect" disabled>Disconnect</button></p>
  <p data-role="status" role="status" aria-live="polite">Connect the repeater, then set its registered key.</p>
  <p data-role="error" role="alert" style="color:#b42318"></p>
</form>

<script>
(() => {
  const root = document.querySelector('[data-role="form"]');
  if (!root) return;
  const keyInput = root.querySelector('[data-role="key"]');
  const status = root.querySelector('[data-role="status"]');
  const error = root.querySelector('[data-role="error"]');
  const connect = root.querySelector('[data-action="connect"]');
  const apply = root.querySelector('[data-action="apply"]');
  const disconnect = root.querySelector('[data-action="disconnect"]');
  let port, reader, writer, readTask, connected = false;
  let expectedPub = '', response = '';
  const enc = new TextEncoder();
  const decoder = new TextDecoder();
  function setStatus(text) { status.textContent = text; }
  async function readLoop() {
    let line = '';
    try {
      while (connected) {
        const { value, done } = await reader.read();
        if (done) break;
        for (const char of decoder.decode(value, { stream: true })) {
          if (char === '\r' || char === '\n') {
            // Drop echoed command lines entirely: they contain the private key.
            if (line.startsWith('set prv.key ')) { line = ''; continue; }
            if (line.includes(expectedPub) && expectedPub) response = 'verified';
            else if (line.startsWith('OK, reboot to apply! New pubkey: ')) response = line.slice('OK, reboot to apply! New pubkey: '.length).trim();
            else if (line === 'Error, bad key') response = 'bad-key';
            line = '';
          } else if (line.length < 180) line += char;
          else line = '';
        }
      }
    } catch (_) { /* Disconnect during reboot is expected. */ }
  }
  async function send(line) { await writer.write(enc.encode(line + '\r')); }
  async function close() {
    connected = false;
    try { if (reader) await reader.cancel(); } catch (_) {}
    try { if (readTask) await readTask; } catch (_) {}
    try { reader?.releaseLock(); } catch (_) {}
    try { writer?.releaseLock(); } catch (_) {}
    try { await port?.close(); } catch (_) {}
    reader = writer = port = null;
    apply.disabled = true; disconnect.disabled = true; connect.disabled = false;
  }
  connect.addEventListener('click', async () => {
    error.textContent = '';
    if (!('serial' in navigator)) { error.textContent = 'Web Serial is unavailable. Use Chrome or Edge on desktop.'; return; }
    try {
      port = await navigator.serial.requestPort();
      await port.open({ baudRate: 115200 });
      reader = port.readable.getReader(); writer = port.writable.getWriter(); connected = true;
      readTask = readLoop();
      connect.disabled = true; disconnect.disabled = false; apply.disabled = false;
      setStatus('USB connected. Paste the matching registered key and choose Set key and reboot.');
    } catch (e) { error.textContent = e.message || 'Could not open USB serial.'; await close(); }
  });
  disconnect.addEventListener('click', close);
  apply.addEventListener('click', async () => {
    error.textContent = '';
    const key = keyInput.value.trim();
    if (!/^[0-9a-fA-F]{128}$/.test(key)) { error.textContent = 'Enter exactly 128 hexadecimal characters.'; return; }
    if (!connected) { error.textContent = 'Connect the repeater over USB first.'; return; }
    expectedPub = ''; response = '';
    apply.disabled = true;
    try {
      setStatus('Applying the private key locally…');
      await send('set prv.key ' + key);
      const deadline = Date.now() + 10000;
      while (Date.now() < deadline && !response) await new Promise(resolve => setTimeout(resolve, 100));
      if (response === 'bad-key') throw new Error('The device rejected this key. Check the keygen backup and firmware version.');
      if (!/^[0-9a-f]{64}$/.test(response)) throw new Error('No public-key confirmation received. The key may not have been applied.');
      expectedPub = response;
      keyInput.value = '';
      await send('reboot');
      setStatus('Key installed. Reconnect after reboot and run get public.key to confirm ' + expectedPub + ' matches the padrón.');
      await close();
    } catch (e) { error.textContent = e.message || 'Key provisioning failed.'; apply.disabled = false; }
  });
  window.addEventListener('pagehide', () => { keyInput.value = ''; void close(); });
})();
</script>

The MeshCore repeater CLI requires local/admin access for `set prv.key`. The device validates the key, stores it, and needs a reboot to use it. Reconnect after reboot and run `get public.key`; confirm it matches the registered public key shown in the keygen backup. The provisioner never prints serial input or writes the private key to a URL, log, or file.
