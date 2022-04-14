# igdb-ts-example-backend
This is an example node.js, Express backend which uses [igdb-ts](https://github.com/AaronWLChan/igdb-ts).

This backend uses [Heroku Postgres](https://devcenter.heroku.com/articles/heroku-postgresql) to persist the Twitch App Access Token.

### Code

```javascript
const express = require("express")
const dotenv = require("dotenv")
const { Pool } = require("pg")
const { IGDB } = require("igdb-ts")

//Load environment variables
dotenv.config();

const PORT = process.env.PORT || 3000;

//Init Express
const app = express();

//Connect to Heroku Postgres
const pool = new Pool({
    connectionString: process.env.DATABASE_URL,
    ssl: { rejectUnauthorized: false }
});

//Create IGDB Wrapper
const igdb = new IGDB();

//CORS - To access Heroku
app.use(function(_, res, next) {
    res.header("Access-Control-Allow-Origin", "*");
    res.header("Access-Control-Allow-Headers", "Origin, X-Requested-With, Content-Type, Accept");
    next();
});

//Simply get the name of a game
app.get("/games", async (_, res) => {

    let game = await igdb.getGames({ fields: ["name"], limit: 1 })

    if (game){
        res.status(200).json(game)
    }

    else {
        console.log("Failed to get game.")
        res.status(500)
    }

});

//Wrap in async function to use await.
(async() => {
    let clientToken = undefined

    //Get existing token
    let result = await pool.query("SELECT token, tokenExpiry FROM tokens;")

    //If exists, use it
    if (result.rows) {
        clientToken = result.rows[0]
        console.log("Using stored token.")
    }

    //Upsert new token
    const onAccessTokenReceived = (token, tokenExpiry) => {
        console.log("Saving new token...")
        
        const UPSERT = `INSERT INTO tokens(id, token, tokenExpiry) 
                        VALUES (1, $1, $2)
                        ON CONFLICT (id) DO UPDATE SET token = $1, tokenExpiry = $2;
                        `

        pool.query(UPSERT, [token, tokenExpiry])
            .then(() => console.log("Saved token to db."))
            .catch(() => console.log("Failed to save token to db."))

    }
 
    //Init IGDB
    await igdb.init(process.env.CLIENT_ID, process.env.CLIENT_SECRET, clientToken, onAccessTokenReceived )
                .then(() => console.log("IGDB inited!"))

    //Listen
    app.listen(PORT, () => {
        console.log("Listening on Port: " + PORT)
    })
    
})();


```

### Helpful Links
* [How to setup Heroku Postgres](https://devcenter.heroku.com/articles/heroku-postgresql).
* [igdb-ts](https://github.com/AaronWLChan/igdb-ts) - IGDB TypeScript Wrapper 
* [How Twitch Authentication works](https://dev.twitch.tv/docs/authentication).
* [IGDB API Documentation](https://api-docs.igdb.com/#about)
