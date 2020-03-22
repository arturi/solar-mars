<script>
	import FeedMe from 'feedme'
	import http from 'http'
	import striptags from 'striptags'
	import subscriptions from '../subscriptions.json'

  // Some RSS feeds can't be loaded in the browser due to CORS security.
  // To get around this, you can use a proxy.
	const CORS_PROXY = 'https://cors-anywhere.herokuapp.com/'
	const DESCRIPTION_LENGTH = 500
	const ALLOWED_TAGS = ['a', 'img', 'strong', 'picture', 'figure', 'figcaption']

	fetch(CORS_PROXY + subscriptions.list[3])
		// Retrieve its body as ReadableStream
		.then(response => response.text())
		.then(body => {
			console.log('FETCHED:', CORS_PROXY + subscriptions.list[3])
			let itemCount = 0
			const feed = {
				items: []
			}

			const parser = new FeedMe(true)
			parser.on('title', (title) => {
				feed.title = title
			})
			parser.on('item', (item) => {
				// console.log(item.title)
				feed.items.push(item)
				itemCount++
				if (itemCount === 5) {
					console.log(feed)
				}
			})
			parser.on('link', (link) => {
				if (typeof link === 'object' && link.rel === 'alternate') {
					feed.link = link.href
				} else if (typeof link === 'string') {
					feed.link = link
				}
			})
			parser.on('home_page_url', (link) => {
				feed.link = parsedFeed.home_page_url
			})

			const onUpdatedDate = (date) => {
				feed.updated = date
			}
			parser.once('pubdate', onUpdatedDate)
			parser.once('lastbuilddate', onUpdatedDate)
			parser.once('updated', onUpdatedDate)
			parser.on('end', () => {
				console.log('_____ENDEND___')
				const parsedFeed = parser.done()
				console.log('PARSER DONE', parsedFeed)
			})
			parser.write(body)
		})

	let allEntries = []

	function getFeedData (url) {
		return new Promise((resolve, reject) => {
			http.get(CORS_PROXY + url, (res) => {
				if (res.statusCode !== 200) {
					const err = new Error(`${url} returned status code ${res.statusCode}`)
					reject(err)
					return
				}
				const parser = new FeedMe(true)
				// parser.on('title', (title) => {
				// 	feed.title = title
				// })
				// parser.on('item', (item) => {
				// 	feed.items.push(item)
				// })
				// Try to get date from one of the fields.
				// parser.once('pubdate', getdate)
				// parser.once('lastbuilddate', getdate)
				// parser.once('updated', getdate)
				parser.on('error', (err) => reject(err))
				parser.on('end', () => {
					console.log(`Finished parsing feed ${url}`)
					const parsedFeed = parser.done()

					const feed = {}
					feed.items = parsedFeed.items.slice(0, 10)
					feed.title = parsedFeed.title
			
					if (parsedFeed.link && typeof parsedFeed.link === 'object' && parsedFeed.link[0] && parsedFeed.link[0].rel === 'alternate') {
						feed.link = parsedFeed.link[0].href
					} else if (parsedFeed.link && typeof parsedFeed.link === 'string') {
						feed.link = parsedFeed.link
					}

					if (parsedFeed.home_page_url) {
						feed.link = parsedFeed.home_page_url
					}

					resolve(feed)
				})
				res.pipe(parser)
			})
		})		
	}
	
	function sortFeedsByDate () {
		allEntries.sort((a, b) => (a.unixTime < b.unixTime) ? 1 : -1)
	}

	function unixToHumanDate (unixTime) {
		const date = new Date(unixTime)
		const year = date.getFullYear()
		const month = date.getMonth()
		const day = date.getDate()
		const hours = date.getHours()
		const minutes = '0' + date.getMinutes()
		const seconds = '0' + date.getSeconds()

 		const humanTimestamp = day + '.' + month + '.' + year + ' ' + hours + ':' + minutes.substr(-2)
		return humanTimestamp
	}

  async function parseFeeds (feedList) {
		console.log('Loading your subscriptions...')
		console.log(feedList)
    feedList.forEach(feedURL => {
			getFeedData(CORS_PROXY + feedURL)
				.then(feed => {
					const currentFeedEntries = parseSingleFeed(feed)
					allEntries = [...allEntries, ...currentFeedEntries]
					sortFeedsByDate()
				})
				.catch(err => console.error(err))
		})
  }

  function parseSingleFeed (feed) {
		const entries = []

    feed.items.forEach(item => {
      const entry = {
				feed: {
					title: feed.title,
					link: feed.link
				},
        title: item.title ? item.title : null,
        pubDate: item.updated || item.pubdate || item.date_published
			}

			if (item.url) {
				entry.link = item.url
			}
	
			if (item.link) {
				entry.link = item.link
			}

			if (item.link && item.link.href) {
				entry.link = item.link.href
			}

			if (Array.isArray(item.link)) {
				item.link.forEach((url) => {
					if (url.rel === 'alternate') {
						entry.link = url.href
					}
				})
			}
			
			if (item.content) {
				entry.excerpt = item.content.substring(0, DESCRIPTION_LENGTH)
			}

			if (item.description) {
				entry.excerpt = item.description.substring(0, DESCRIPTION_LENGTH)
			}

			if (item.content_html) {
				entry.excerpt = item.content_html.substring(0, DESCRIPTION_LENGTH)
			}

			if (entry.excerpt) {
				entry.excerpt = entry.excerpt.replace(/style="[^"]*"/, '')
				entry.excerpt = striptags(entry.excerpt, ALLOWED_TAGS);
			}

			entry.unixTime = new Date(entry.pubDate).valueOf()
			entry.humanTimestamp = new Date(entry.pubDate)

			entries.push(entry)
		})

		// console.log(entries)

		return entries
  }

	// parseFeeds(subscriptions.list)
</script>

<style>
  main {
    font-family: 'Avenir Next', Avenir, '-apple-system', Helvetica, Arial, sans-serif;
		max-width: 600px;
		margin: auto;
  }

	.solar-mars-title {
		margin-top: 20px;
		margin-bottom: 50px;
		color: var(--hot-pink);
	}

	.solar-mars-title svg {
		vertical-align: text-bottom;
		display: inline-block;
		width: 50px;
		height: 50px;
		fill: currentColor;
	}

	article {
		margin-bottom: 70px;
	}

	article header {
		margin-bottom: 15px;
	}

	article header h2 {
		font-size: 16px;
		font-weight: normal;
		margin-bottom: 2px;
	}

	article header h2 a {
		color: var( --mid-gray);
		text-decoration: none;
	}

	article header h1 {
		font-size: 20px;
	}

	article header h1 a {
		color: var(--dark-gray);
		text-decoration: none;
	}

	.entry-content {
		font-size: 18px;
		margin-bottom: 15px;
	}

	.entry-content :global(a) {
		color: var(--pink);
	}

	.entry-content :global(img) {
		margin: 1em 0;
	}

	date {
		display: block;
		font-size: 12px;
		font-weight: 500;
	}

	date a {
		color: var(--gray);
		text-decoration: none;
	}
</style>

<main>
  <h1 class="solar-mars-title">
		<svg xmlns="http://www.w3.org/2000/svg" width="150" height="190" viewBox="0 0 150 190"><path fill-rule="evenodd" clip-rule="evenodd" d="M119.67 66.736c-1.361.905-33.158 21.516-33.158 21.516L74.604 71.437s31.613-21.809 32.905-22.676c7.204-4.835 18.293 13.904 12.161 17.975zm-3.955-10.123c-3.782-6.552-9.222-3.529-5.192 3.45 3.681 6.375 8.96 3.077 5.192-3.45zM72.738 75.906l7.202 13.899-23.913 16.6-8.864-12.284 4.894-3.486-3.692-3.924-6.379 3.536-5.382-7.543 25.67-17.126 4.803 7.35-8.524 6.319 2.705 4.836 11.48-8.177zM46.12 99.673l3.755 7.549-13.738 9.784-7.607-10.036 5.447-4.215 4.505 2.888 7.638-5.97zM81.962 96.6l24.28 48.266-5.32 2.459-17.674-29.847-11.698.668-18.692 28.868-5.164-1.61 21.241-41.075L81.962 96.6zm-1.577 50.456h-6.676l-.994-22.955h9.346l-1.676 22.955z"/></svg>
		Solar Mars
	</h1>

	{#each allEntries as entry}
		<article>
			<header>
				{#if entry.feed.title}
					<h2><a href="{entry.feed.link}">{entry.feed.title}</a></h2>
				{/if}
				{#if entry.title}
					<h1><a href="{entry.link}">{entry.title}</a></h1>
				{/if}
			</header>
			<div class="entry-content">
				{@html entry.excerpt}...
			</div>
			<date>
				<a class="pink" href="{entry.link}">→ {unixToHumanDate(entry.pubDate)}</a>
			</date>
		</article>
	{:else}
		Loading...
	{/each}
</main>
