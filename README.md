# wyr-api

a small, silly **would-you-rather API** written in [PlainScript](https://github.com/ayoistooslick/plainscript) and running on Express.

there are good questions, spicy questions, and 18+ questions. no database, no login, no drama. just grab a question and argue about it with your friends.

## live API

**https://wyr-api-vhz0.onrender.com**

## quick start

```bash
curl https://wyr-api-vhz0.onrender.com/api/wyr
```

no `level` means the API randomly picks `good`, `spicy`, or `18+`. if you want to keep things wholesome, pass a level:

```bash
curl "https://wyr-api-vhz0.onrender.com/api/wyr?level=good"
curl "https://wyr-api-vhz0.onrender.com/api/wyr?level=spicy"
curl "https://wyr-api-vhz0.onrender.com/api/wyr?level=18%2B"
```

## endpoints

| method | endpoint | what it does |
| --- | --- | --- |
| `GET` | `/` | tiny intro and a usage hint |
| `GET` | `/api/wyr` | returns one random question |
| `GET` | `/api/wyr?level=good` | returns one good question |
| `GET` | `/api/wyr?level=spicy` | returns one spicy question |
| `GET` | `/api/wyr?level=18%2B` | returns one 18+ question |
| `GET` | `/api/levels` | lists the supported levels |
| `GET` | `/api/health` | basic health check and question count |

## response shape

```json
{
  "id": "good-7",
  "level": "good",
  "question": "Would you rather be famous for something good or quietly successful?",
  "optionA": "famous for good",
  "optionB": "quietly successful"
}
```

## level rules

- `good` = safe-ish questions for basically everyone
- `spicy` = flirty and a little awkward
- `18+` = adults only, use your common sense
- no level = random level
- anything else = `400` with a small error response

## run it locally

```bash
git clone https://github.com/ayoistooslick/wyr-api.git
cd wyr-api
npm install
npm run build
npm start
```

then open <http://localhost:3000/api/wyr>. Render sets `PORT` for us, and the app uses it.

## how it is made

The source lives in [`src/app.pln`](./src/app.pln). PlainScript compiles the `web app` and route blocks into a regular Express app. The question bank is just a big list of records because honestly that is enough for this little project.

Want to add a question? add another record to the matching list in `src/app.pln`, run `npm run build`, and push. Render auto-deploys from `main`.

## tiny disclaimer

This is a hobby API, so the questions are subjective and the list is finite, even if it is trying very hard to feel infinite. Please don't use the 18+ set around people who didn't ask for it.
