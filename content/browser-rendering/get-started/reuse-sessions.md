---
pcx_content_type: get-started
title: Reuse sessions
weight: 3
---

# Reuse sessions

The best way to improve the performance of your browser rendering worker is to reuse sessions. One way to do that is via [Durable Objects](../browser-rendering-with-do/). Another way is to keep the browser open after you've finished with it, making it available for the next request.

In short, this entails using `browser.disconnect()` instead of `browser.close()`, and, if there are available sessions, using `puppeteer.connect(env.MY_BROWSER, sessionID)` instead of launching a new browser session.

## 1. Create a Worker project

[Cloudflare Workers](/workers/) provides a serverless execution environment that allows you to create new applications or augment existing ones without configuring or maintaining infrastructure. Your Worker application is a container to interact with a headless browser to do actions, such as taking screenshots.

Create a new Worker project named `browser-worker` by running:

{{<tabs labels="npm | yarn">}}
{{<tab label="npm" default="true">}}

```sh
$ npm create cloudflare@latest
```
{{</tab>}}
{{<tab label="yarn">}}

```sh
$ yarn create cloudflare
```
{{</tab>}}
{{</tabs>}}

## 2. Install Puppeteer

In your `browser-worker` directory, install Cloudflare’s [fork of Puppeteer](/browser-rendering/platform/puppeteer/):

```sh
$ npm install @cloudflare/puppeteer --save-dev
```

## 3. Configure `wrangler.toml`

```
name = "browser-worker"
main = "src/index.ts"
compatibility_date = "2023-03-14"
compatibility_flags = [ "nodejs_compat" ]

browser = { binding = "MYBROWSER" }
```

## 3. Code

The script below starts by fetching the current running sessions. If there are any that don't already have a worker connection, it picks a random session ID and attempts to connect (`puppeteer.connect(..)`) to it. If that fails or there were no running sessions to start with, it launches a new browser session (`puppeteer.launch(..)`). Then, it goes to the website and fetches the dom. Once that's done, it disconnects (`browser.disconnect()`), making the connection available to other workers.

```ts
import puppeteer from "@cloudflare/puppeteer";

interface Env {
	MYBROWSER: Fetcher;
}

export default {
	async fetch(request: Request, env: Env) {
		const url = new URL(request.url);
		let reqUrl = url.searchParams.get("url") || 'https://example.com';
		reqUrl = new URL(reqUrl).toString(); // normalize

		// Pick random session from open sessions
		let sessionId = await this.getRandomSession(env.MYBROWSER)
		let browser, launched
		if (sessionId) {
			try {
				browser = await puppeteer.connect(env.MYBROWSER, sessionId)
			} catch (e) {
				// another worker may have connected first
				console.log(`Failed to connect to ${sessionId}. Error ${e}`)
			}
		}
		if (!browser) {
			// No open sessions, launch new session
			browser = await puppeteer.launch(env.MYBROWSER)
			launched = true
		}

		sessionId = browser.sessionId() // get current session id

		// Do your work here
		const page = await browser.newPage();
		const response = await page.goto(reqUrl);
		const html = await response!.text()

		// All work done, so free connection (IMPORTANT!)
		await browser.disconnect()

		return new Response(`${launched ? 'Launched' : 'Connected to'} ${sessionId} \n-----\n` + html, {
			headers: {
				"content-type": "text/plain",
			}
		})
	},

	// Pick random free session
	// Other custom logic could be used instead
	async getRandomSession(endpoint: puppeteer.BrowserWorker): Promise<string> {
		const sessions: puppeteer.ActiveSession[] = await puppeteer.sessions(endpoint);
		console.log(`Sessions: ${JSON.stringify(sessions)}`)
		const sessionsIds = sessions
			.filter(v => {
				return !v.connectionId; // remove sessions with workers connected to them
			})
			.map(v => {
				return v.sessionId;
			});
		if (sessionsIds.length === 0) {
			return
		}

		const sessionId = sessionsIds[Math.floor(Math.random() * sessionsIds.length)];

		return sessionId!;
	}
}
```

Besides `puppeteer.sessions()`, we've added other methods to facilitate session management - check them out [here](../../platform/puppeteer/#session-management).

## 6. Test

Run `npx wrangler dev --remote` to test your Worker locally before deploying to Cloudflare's global network.

To test go to the following URL: 

`<LOCAL_HOST_URL>/?url=https://example.com`

## 7. Deploy

Run `npx wrangler deploy` to deploy your Worker to the Cloudflare global network and then to go to the following URL:

`<YOUR_WORKER>.<YOUR_SUBDOMAIN>.workers.dev/?url=https://example.com`