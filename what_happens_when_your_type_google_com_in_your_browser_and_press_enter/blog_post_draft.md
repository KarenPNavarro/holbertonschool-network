# What Happens When You Type https://www.google.com and Press Enter

It is the most ordinary thing a person can do with a computer. You click the address bar, type a name you have typed a thousand times, and hit Enter. A moment later, a page appears.

That moment is not simple. Between your keystroke and the first pixel, your request crosses a naming system older than the web, negotiates an encrypted tunnel with a stranger, passes at least one checkpoint designed to turn it away, and gets handed between several machines that each do one job well. Here is what actually happens.

## 1. The name has to become a number

Computers do not route traffic to names. They route it to IP addresses. So the first job is translating `www.google.com` into something like `142.250.80.100`. That translation is DNS, the Domain Name System.

Your browser does not go straight to the internet to ask. It works through a series of caches, nearest first, because the fastest lookup is the one that never leaves your machine:

1. **The browser's own cache.** It remembers recent lookups.
2. **The operating system cache**, and the `hosts` file. This is a plain text file that maps names to addresses, and it wins before any network lookup happens. Edit it and you can make `facebook.com` point anywhere you like, which is exactly how a lot of ad blockers work.
3. **Your resolver**, usually run by your ISP or a public service like `8.8.8.8`.

If the resolver has no cached answer, it starts walking the hierarchy. It asks a **root server**, which does not know the address but knows who handles `.com`. It asks that **TLD server**, which does not know either but knows Google's **authoritative name server**. That last one holds the real record and returns the address. The resolver caches the answer for the duration of its TTL, so the next person asking gets it instantly.

Four questions to answer one. This is why DNS is a favorite interview topic: it is the step most people skip, and it is the step that explains most "the site is down for me but not for you" mysteries.

## 2. Opening the connection: TCP/IP

Now your browser has an address. It needs a conversation.

**IP** handles addressing and delivery: it gets packets from your machine to Google's, hopping through routers along the way. IP alone makes no promises, though. Packets can arrive out of order, duplicated, or not at all.

**TCP** is the layer that makes the connection reliable. Before any data moves, the two machines perform a **three-way handshake**:

- Your machine sends a `SYN` — "I would like to talk."
- The server replies `SYN-ACK` — "Heard you, and I would like to talk too."
- Your machine sends `ACK` — "Confirmed."

From then on, TCP numbers every segment, acknowledges what arrives, and retransmits what does not. That is the tradeoff: TCP is slower than UDP, which fires packets off and never looks back, but your bank balance and this web page both need to arrive intact and in order. The connection targets **port 443**, the standard port for HTTPS. If you had typed `http://`, it would be port 80.

## 3. The firewall decides whether you get through

Firewalls sit on the path and filter traffic against a ruleset: which ports, which protocols, which source addresses are allowed.

There is one on your side — on your machine, your router, and possibly your company's network — governing what you are permitted to reach. There are more on Google's side, and theirs are the stricter ones. A well-configured server firewall exposes only the ports that must be public, typically 443 and 80, and silently drops everything else. Port 22 for SSH might be open, but only to a specific range of administrator IPs.

This is the layer that decides whether your packets are even allowed to continue. When a request hangs with no response at all rather than returning an error, a firewall dropping packets is usually why.

## 4. HTTPS and SSL/TLS: making the conversation private

The `https` you typed means the connection gets encrypted before any real data crosses it. Immediately after the TCP handshake comes the **TLS handshake** (still often called SSL, after the protocol it replaced).

Three things happen:

- **The server proves who it is.** It presents a certificate, issued by a Certificate Authority your browser already trusts. Your browser checks that it was issued by a legitimate CA, that it has not expired, and crucially that it was issued for `www.google.com`. This is what stops someone who has hijacked your DNS from impersonating Google — they can send you to their server, but they cannot produce a valid certificate for a domain they do not control.
- **The two sides agree on encryption.** They negotiate a cipher and exchange keys, ending up with a shared session key only they possess.
- **Everything afterward is encrypted** with that key.

Anyone watching your traffic now sees that you contacted a Google IP address, and nothing more — not the page you requested, not what came back. The padlock in the address bar means this succeeded.

Only now does your browser send the actual request: `GET / HTTP/1.1`.

## 5. The load balancer picks a server

Your request does not reach "the Google server," because no single machine could serve Google. It reaches a **load balancer**, which stands in front of a pool of servers and distributes incoming traffic across them.

How it chooses varies. **Round-robin** rotates through servers in turn. **Least-connections** sends you to whichever is handling the least work. **IP hash** routes based on your address, so you consistently land on the same server — which matters when sessions are stored locally.

The load balancer also runs **health checks**, pinging each server on an interval. When one stops responding, it is pulled from the pool and traffic silently routes around it. This is what allows a service to lose a machine at 3 a.m. without anyone noticing, and what allows deploys to happen with no downtime: take servers out of rotation a few at a time, update them, put them back.

## 6. Web server, application server, database

Behind the load balancer, the work splits across three distinct roles.

The **web server** — Nginx or Apache — receives the HTTP request. For static content such as images, CSS, and JavaScript files, it answers immediately from disk. There is no reason to involve anything more complicated to hand back a logo.

Anything that has to be *computed* gets passed to the **application server**, where the actual code runs. This is where your search query is parsed, your session is checked, your permissions are evaluated, and the response is assembled. The web server handles delivery; the application server handles logic.

When the application needs stored data — user accounts, search indexes, preferences — it queries the **database**. At scale this is rarely one machine either: a primary handles writes while replicas serve reads, and caches like Redis sit in front to keep frequently requested data in memory rather than hitting disk.

The data flows back up the same chain: database to application server, which renders a response, to web server, to load balancer, back through the encrypted tunnel to your browser.

## 7. The browser draws the page

Your browser receives HTML and starts parsing it into the DOM, requesting the CSS, JavaScript, and images it references as it goes — each one potentially a new round trip. It applies styles, runs scripts, calculates layout, and paints.

Then the page is there, and it felt instant.

## Why this question gets asked

Every one of these layers is a place things break, and knowing the chain is what lets you reason about failures instead of guessing. A certificate error is TLS. A request that hangs forever with no response is usually a firewall. A site that works for your colleague but not for you is often DNS caching. A slow page that is fast on a second load is a caching layer doing its job.

Next time a page takes a beat too long to appear, you will have a decent idea of which of these steps is the one keeping you waiting.
