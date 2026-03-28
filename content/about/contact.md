---
title: Contact
toc: false
cascade:
  type: docs
previous: support
next: events
weight: 5
breadcrumbs: false
---
<body>

   <h1>Contact</h1>
   <p>Send a message.</p>

   <form id="contact-form" onsubmit="return false;">
       <div class="form-group">
           <label for="name">Name</label>
           <input type="text" id="name" name="name" required>
       </div>
       
       <div class="form-group">
           <label for="contact-info">Contact Info</label>
           <input type="text" id="contact-info" name="contact-info" 
               placeholder="email@example.com" required>
           <small>How you want to be contacted back.</small>
       </div>
       
       <div class="form-group">
           <label for="message">Comments</label>
           <textarea id="message" name="message" rows="5" placeholder="Your message..." required></textarea>
       </div>
       
       <button type="submit" id="submit-btn">Send Message</button>
   </form>

   <div id="status"></div>

   <script src="https://unpkg.com/nostr-tools@1.17.0/lib/nostr.bundle.js"></script>
   <script>
       // CONFIGURATION: Replace these with your own values
       const RECIPIENT_NPUB = 'npub1c0r3ytrr4afgrlhrhyec6y9wvkckdllx7ul3cfevtsgjqcrhx8tsdzqs7w';
       const RELAYS = [
		'wss://bits.sdbitcoiners.com/nostrrelay/Leug2TNY',
		'wss://relay.damus.io',
		'wss://nos.lol',
		'wss://relay.nostr.band'
           	// Add more relays if desired
       ];

       // Decode npub to hex pubkey
       let RECIPIENT_HEX;
       try {
           const decoded = window.NostrTools.nip19.decode(RECIPIENT_NPUB);
           RECIPIENT_HEX = decoded.data;
       } catch (e) {
           console.error('Invalid npub in configuration');
       }

       document.getElementById('contact-form').addEventListener('submit', async (e) => {
           e.preventDefault();
           
           const submitBtn = document.getElementById('submit-btn');
           const status = document.getElementById('status');
           
           if (!RECIPIENT_HEX) {
               status.innerHTML = '<strong>Configuration error:</strong> Invalid npub.';
               status.className = 'error';
               return;
           }
           
           submitBtn.disabled = true;
           submitBtn.textContent = 'Sending...';
           status.className = '';
           status.style.display = 'none';
           
           try {
               const sk = window.NostrTools.generatePrivateKey();
               const pk = window.NostrTools.getPublicKey(sk);
               
               const formData = {
                   name: document.getElementById('name').value,
                   contact: document.getElementById('contact-info').value,
                   message: document.getElementById('message').value,
                   timestamp: new Date().toISOString(),
                   source: 'Contact Form'
               };
               
               const encryptedContent = await window.NostrTools.nip04.encrypt(
                   sk, 
                   RECIPIENT_HEX, 
                   JSON.stringify(formData)
               );
               
               const event = {
                   kind: 4,
                   pubkey: pk,
                   created_at: Math.floor(Date.now() / 1000),
                   tags: [['p', RECIPIENT_HEX]],
                   content: encryptedContent
               };
               
               event.id = window.NostrTools.getEventHash(event);
               event.sig = window.NostrTools.signEvent(event, sk);
               
               let publishedCount = 0;
               const publishPromises = RELAYS.map(relayUrl => {
                   return new Promise((resolve) => {
                       const ws = new WebSocket(relayUrl);
                       let timeout;
                       
                       ws.onopen = () => {
                           ws.send(JSON.stringify(['EVENT', event]));
                           timeout = setTimeout(() => {
                               ws.close();
                               resolve(false);
                           }, 5000);
                       };
                       
                       ws.onmessage = (msg) => {
                           const data = JSON.parse(msg.data);
                           if (data[0] === 'OK' && data[1] === event.id) {
                               clearTimeout(timeout);
                               ws.close();
                               resolve(true);
                           }
                       };
                       
                       ws.onerror = () => {
                           clearTimeout(timeout);
                           ws.close();
                           resolve(false);
                       };
                   });
               });
               
               const results = await Promise.all(publishPromises);
               publishedCount = results.filter(r => r).length;
               
               if (publishedCount > 0) {
                   status.innerHTML = '<strong>Message sent!</strong><br>Published to ' + publishedCount + ' relay(s).';
                   status.className = 'success';
                   document.getElementById('contact-form').reset();
               } else {
                   throw new Error('No relays accepted the message');
               }
               
           } catch (error) {
               console.error(error);
               status.innerHTML = '<strong>Network error.</strong><br>Please try again later.';
               status.className = 'error';
           } finally {
               submitBtn.disabled = false;
               submitBtn.textContent = 'Send Message';
           }
       });
   </script>

</body>
