---
layout: post
title: "Vestaboard"
description: "A morning weather and tide briefing, plus Slack and Discord bots, for my Vestaboard."
tags: [javascript, nodejs, integrations]
---
A [Vestaboard](https://www.vestaboard.com/) is a split-flap display with 6 rows of 22 characters (and they're steadily growing with other sizes and options). The company brought this this classic "train station" display to the modern world with API support. I got one as a huge gift (thank you babydoll 🥰) and immediately started building integrations with it.

I wrote three small Node.js services for mine: a scheduler that posts a morning briefing, plus Slack and Discord bots that let other people post messages to it.

- [Scheduler Source Code on github](https://github.com/aarace/vestabot-scheduler)
- [Slack Integration Source Code on github](https://github.com/aarace/vestabot-slack)
- [Discord Integration Source Code on github](https://github.com/aarace/vestabot-discord)

All three use the same setup: a Vestaboard API token in a `.env` file, then a single `node index.js` command, or a Docker container with `--restart unless-stopped` so it keeps running.

## Morning Briefing (Scheduler)

Every morning at 6:30 the scheduler posts the date, weather, sunrise and sunset, and the day's tides (I live in a beach town):

```
     MONDAY JUNE 22ND
  PARTLY CLOUDY 72°/58°
 RISE 5:12AM SET 8:34PM
HIGH               LOW
10:23AM        3:45PM
 4:12PM        9:58PM
```

Both data sources are free and need no API key. Weather comes from [Open-Meteo](https://open-meteo.com/), and tides come from [NOAA Tides & Currents](https://tidesandcurrents.noaa.gov/api-guide.html). The tide station, location, time zone, and schedule are all set in the `.env` file, and they default to Cohasset Harbor. The scheduler uses [croner](https://github.com/hexagon/croner) for the timing:

```js
new Cron(MORNING_CRON, { timezone: TIMEZONE }, runMorningBriefing);
```

Both APIs are called at the same time, and their results are combined into the grid:

```js
const [weather, tides] = await Promise.all([getCurrentWeather(), getTodayTides()]);
const grid = buildMorningGrid(weather, tides);
await postGridToVestaboard(grid);
```

Pass `--now` to post a briefing right away, which is handy for testing.

### Laying out the board

Plain text sent to the Vestaboard API gets laid out by Vestaboard, so you don't control where anything goes. To line up columns like the tide table, you send a 6×22 array of character codes instead. Each character on the board has a numeric code:

```js
const CHAR_CODES = {
  ' ':  0,
  A:1,  B:2,  C:3,  /* ... */  Z:26,
  '1':27, '2':28, /* ... */  '0':36,
  '!':37, '@':38, /* ... */  '°':62,
};

function textRow(text, width = 22) {
  const codes = [...text.slice(0, width)].map(charCode);
  while (codes.length < width) codes.push(BLANK);
  return codes;
}
```

From there, each row is built with ordinary string padding. Here, high tide times are left-aligned and low tide times are right-aligned:

```js
function tideTimeRow(high, low) {
  const h = (high ? formatTideTime(high.time) : '').padEnd(13).slice(0, 13);
  const l = (low  ? formatTideTime(low.time)  : '').padStart(9).slice(0, 9);
  return textRow(h + l);
}
```

Open-Meteo reports conditions as [WMO weather codes](https://open-meteo.com/en/docs#weathervariables). These get mapped to short labels like `PARTLY CLOUDY` or `TSTORM+HAIL`, so the conditions and temperatures fit on one 22-character line.

## Slack

The Slack app adds two slash commands:

- `/vesta [message]` posts any text to the board
- `/tides` posts the day's high and low tides as a grid, built the same way as the scheduler's

It uses [Bolt](https://slack.dev/bolt-js/) in Socket Mode, which means it connects out to Slack. There's no public URL to host and nothing to open on the firewall, so it can run on any machine at home.

To keep the board from getting spammed, each user can post once per minute. If someone tries again too soon, they get a private reply only they can see:

```js
const lastPost = lastPostTime.get(userId);
if (lastPost && now - lastPost < RATE_LIMIT_MS) {
  const secondsLeft = Math.ceil((RATE_LIMIT_MS - (now - lastPost)) / 1000);
  await respond({
    response_type: 'ephemeral',
    text: `⏳ You can only post once per minute. Please wait ${secondsLeft} more second${secondsLeft === 1 ? '' : 's'}.`,
  });
  return;
}
```

When a post works, the bot replies in the channel so everyone can see who posted what.

## Discord

The Discord bot does the same thing for a Discord server with a `/vesta` slash command and the same one-post-per-minute limit. It registers the slash command itself when it starts up. Because the call to the Vestaboard API can take a moment, the bot defers its reply first. That tells Discord the bot is working on it, so the interaction doesn't time out:

```js
await interaction.deferReply();
// ... post to the Vestaboard ...
await interaction.editReply(`Posted to the Vestaboard: **${text}**`);
```
