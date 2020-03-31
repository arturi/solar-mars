<script>
	import FeedMe from 'feedme'
	import striptags from 'striptags'
	import subscriptions from '../subscriptions.json'

  // Some RSS feeds can't be loaded in the browser due to CORS security.
  // To get around this, we use a proxy
	const CORS_PROXY = 'https://cors-anywhere.herokuapp.com/'
	const DESCRIPTION_LENGTH = 500
	const ALLOWED_TAGS = ['a', 'img', 'strong', 'picture', 'figure', 'figcaption']

	let allEntries = []

	const parseFeedDescription = (str, baseUrl) => {
		const excerpt = str.substring(0, DESCRIPTION_LENGTH)
		const tempNode = document.createElement('div')
		tempNode.innerHTML = excerpt

		const allElements = Array.from(tempNode.querySelectorAll('*'))
		allElements.forEach(el => {
			el.removeAttribute('style')
			el.removeAttribute('class')
		})

		const imageElements = Array.from(tempNode.querySelectorAll('img'))
		imageElements.forEach(el => {
			const isRelativeUrl = new RegExp('^(?:[a-z]+:)?//', 'i')
			const imageSrc = el.getAttribute('src')
			if (!isRelativeUrl.test(imageSrc)) {
				const absoluteUrl = new URL(imageSrc, baseUrl)
				el.src = absoluteUrl.href
			}
		})

		let parsedString = tempNode.innerHTML
		parsedString = striptags(parsedString, ALLOWED_TAGS)
		// parsedString = typograf.execute(parsedString)

		if (str.length > DESCRIPTION_LENGTH) {
			parsedString += '...'
		}

		return parsedString
	}

	function getFeed (url) {
		return new Promise((resolve, reject) => {
			fetch(CORS_PROXY + url)
				.then(response => {
					if (response.ok && response.status === 200) {
						return response.text()
					}
					const err = new Error(`${url} returned status code ${response.status}`)
					throw err
				})
				.then(feedString => {
					let itemCount = 0
					const feed = {
						items: []
					}
					const parser = new FeedMe()
					parser.on('error', (err) => reject(err))
					parser.on('title', (title) => {
						feed.title = title
					})
					parser.on('item', (item) => {
						// Stop parsing once we have 5 items
						if (itemCount === 5) {
							resolve(feed)
						} else {
							feed.items = [item, ...feed.items]
							itemCount++
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
						feed.link = link
					})

					const onUpdatedDate = (date) => {
						feed.updated = date
					}
					parser.once('pubdate', onUpdatedDate)
					parser.once('lastbuilddate', onUpdatedDate)
					parser.once('updated', onUpdatedDate)

					parser.write(feedString)
			})
		})
	}
	
	function sortFeedsByDate (feedArray) {
		return [...feedArray].sort((a, b) => (a.unixTime < b.unixTime) ? 1 : -1)
	}

	function unixToHumanDate (unixTime) {
		const date = new Date(unixTime)
		const year = date.getFullYear()
		const month = date.getMonth() + 1
		const day = date.getDate()
		const hours = date.getHours()
		const minutes = '0' + date.getMinutes()
		const seconds = '0' + date.getSeconds()

 		const humanTime = day + '.' + month + '.' + year + ' ' + hours + ':' + minutes.substr(-2)
		return humanTime
	}

	function timeAgo (dateParam) {
    const date = typeof dateParam === 'object' ? dateParam : new Date(dateParam)
    const today = new Date()
    const yesterday = new Date(today - 86400000)
    const seconds = Math.round((today - date) / 1000)
    const minutes = Math.round(seconds / 60)
    const isToday = today.toDateString() === date.toDateString()
    const isYesterday = yesterday.toDateString() === date.toDateString()

    if (seconds < 5) {
      return 'just now'
    } else if (seconds < 60) {
      return `${seconds} seconds ago`
    } else if (seconds < 90) {
      return 'a minute ago'
    } else if (minutes < 60) {
      return `${minutes} minutes ago`
    } else if (isToday) {
      return `${Math.floor(minutes / 60)} hours ago`
    } else if (isYesterday) {
      return 'yesterday'
    }

    return `${Math.floor(minutes / 1440)} days ago`
  }

  async function parseFeeds (feedList) {
		console.log('Loading your subscriptions...')
		console.log(feedList)
    feedList.forEach((feedURL) => {
			getFeed(CORS_PROXY + feedURL)
				.then(feed => {
					const currentFeedEntries = parseSingleFeed(feed)
					allEntries = sortFeedsByDate(
						[...allEntries, ...currentFeedEntries]
					)
				})
				.catch(err => console.error(err))
		})
  }

  function parseSingleFeed (feed) {
		console.log('Parsing:', feed.title)
		const entries = []

    feed.items.forEach(item => {
      const entry = {
				feed: {
					title: feed.title,
					link: feed.link
				},
        title: item.title || null,
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

			if (item.description) {
				entry.excerpt = parseFeedDescription(item.description, feed.link)
			}
			
			if (item.content) {
				entry.excerpt = parseFeedDescription(item.content, feed.link)
			}

			if (item['content:encoded']) {
				entry.excerpt = parseFeedDescription(item['content:encoded'], feed.link)
			}

			if (item.content_html) {
				entry.excerpt = parseFeedDescription(item.content_html, feed.link)
			}

			entry.unixTime = new Date(entry.pubDate).valueOf()
			entry.humanTime = new Date(entry.pubDate)

			entries.push(entry)
		})

		return entries
  }

	parseFeeds(subscriptions.list)
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
	}

	.solar-mars-title img {
		display: inline-block;
		width: 30px;
		height: 30px;
		fill: currentColor;
	}

	.solar-mars-title a {
		color: var(--hot-pink);
		text-decoration: none;
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

	.entry-content :global(strong) {
		font-weight: 400;
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
		<a href="/">
			<img alt="Solar Mars logo: a pink telescope" src="/favicon.svg">
			Solar Mars
		</a>
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
				{@html entry.excerpt}
			</div>
			<date>
				<a class="pink" href="{entry.link}">→ {timeAgo(entry.pubDate)} ({unixToHumanDate(entry.pubDate)})</a>
			</date>
		</article>
	{:else}
		Loading...
	{/each}
</main>
