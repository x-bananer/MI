# Guide: how to make the menu

Your task: make `GET /api/v1/dishes` so the frontend can get the list of dishes from the database.

## 1. First look at where the data comes from

Open the file:

`backend/src/models/db/schemas/02_tables.sql`

Find the `dishes` table.

Look at which fields it has. Spoiler:
- `id`
- `name`
- `description`
- `price`
- `is_available`
- `created_at`

Then open the file:

`backend/src/models/db/schemas/04_dummy_data.sql`

Look at which dishes are already in the database in the `dishes` table.

## 2. Now create a function to get this data from the database

Go to the folder:

`backend/src/models/db/repositories/`

Create a folder inside it:

`dish`

Inside the `dish` folder create a file:

`dish.repository.js`

In this file you need to write and export a function that goes to our database, takes all dishes from the `dishes` table, and returns them to us. (You need to export this function so it can be used later in the controller. A controller is a file that receives a request from the frontend and sends data back in the response. It's not essential now, we will come back to controllers later.)

How do you write your function?

1. import `select` function

```js
import { select } from '../../db.js';
```

`select` is a select is a helper function created by Luara for reading data from our database. It takes two arguments: an SQL query string and optional extra dynamic parameters. You do not need the parameters yet.

You can find the code of the `select` is here: `backend/src/models/db/db.js` (you do not need to look). In this file Laura created the connection to our database once, set everything up there (I did not even look myself), and exported the `select` function for us, which we can now use anywhere to get data from our database.

*(NOT)FUN FACT: This file actually has two functions: `select` and `execute`. We use `select` when we just want to read data from the database, and we use `execute` when we want to update existing data or add new data. Right now you only need `select`.*

2. use `select` function

Let's try using `select` by creating the `getDishes` function.

- Right below your import, create an exported async function `getDishes`.

```js
export async function getDishes() {
  // ...
};
```

- Inside the `getDishes` function call `select()` and pass it an SQL query that selects the fields id, name, description, price, is_available, created_at from the `dishes` table.

*NOTE: working with a database is an asynchronous operation, so your function must be async, and before select you need to use await.*

```js
select(`
  SELECT id, name, description, price, is_available, created_at
  FROM dishes
`)
```

- Save the result of `select` into a variable called `rows` using `await`.

```js
const rows = await select(...);
```

- Return `rows` from the function.
```js
return rows;
```

All together:

```js
import { select } from '../../db.js';

export async function getDishes() {
  const rows = await select(`
    SELECT id, name, description, price, is_available, created_at
    FROM dishes
  `);

  return rows;
}
```

If your file looks the same, congrats, the main and most important method of your repository is ready.

## 3. Now create a function to transform database data into a nicer shape

Now you need to make a service. A service is an in-between file between the repository and the controller. A repository knows how to get data from the database, a controller knows how to send a response to the frontend, and a service stands between them and transforms ugly raw database data into nice frontend data.

Go to the folder:

`backend/src/services/`

Create a folder inside it:

`dish`

Inside it create a file:

`dish.service.js`

In this file you need to:

- import your repository

```js
import * as dishRepo from '../../models/db/repositories/dish/dish.repository.js';
```

- write and export a function for getting dishes

```js
import * as dishRepo from '../../models/db/repositories/dish/dish.repository.js';

export async function getDishes() {
  const dishes = await dishRepo.getDishes();
  return dishes;
}
```

And where is the transformation into a nicer shape, you may ask? And you would be completely right.

Your service does nothing. For you it is completely useless. You are just passing the data further.

But! Services are needed for other entities, for example carts and orders, where the data is more complex and needs to be collected and transformed. There you cannot do without a separate service file.

With the menu everything is simpler, so here the service is not functionally necessary, you are just following the project structure.

Done? Then your service is ready!

## 4. Now write the controller for your service

We already figured out that:

- the `repository` goes to the database and gets data from it
- the `service` gets data from the `repository` and can transform it into something nicer

And the `controller` is needed for the following:

- receive the request for data from the frontend
- call the `service` to get this data from the repository and therefore from the database
- send this data back to the frontend

So a `controller` is the place where we answer the frontend request.

### How the request arrives from the frontend

When the frontend wants to get the menu, it sends a fetch request to:

```http
/api/v1/dishes
```

This request comes to our backend and goes into the routes file `backend/src/routes.js`. Open it.

This is a pretty scary file. Laura created it. There is a line there:

```js
router.get('/dishes', dishController.list);
```

This means:

- when the frontend sends a request to `GET /api/v1/dishes`
- Express calls the `list` function from the `dishController` file

For you this means that in `dish.controller.js` you need to create this `list` function.

### How to write the `list` function

1. Open the file:

`backend/src/controllers/dish.controller.js`

This is also a pretty scary file. Laura created this one in advance too. Right now this file has temporary placeholders instead of real functions.

2. Import your `service` at the top of the file:

```js
import * as dishService from '../services/dish/dish.service.js';
```

Find the line with the function:

```js
export const list = placeholder('dishes.list');
```

and delete it.

In the place of the deleted function write the beginning of your own function:

```js
export async function list(req, res) {
  // ... Your code will be here
}
```

What is happening here:

- `export` — exports the function so it can be used from other files
- `async` — because inside we will go to the database (controller -> service -> repository -> `select()`), and that is asynchronous, so we need `await`
- `list` — this is the function name the route is already waiting for (Laura decided that name)
- `req` — everything that came from the frontend (`params`, `query`, `body`, `headers`, and so on, this is part of Express)
- `res` — the future response object, which contains different methods, Express gives it to us too

### What to write inside your `list` function

You need to do 3 things:

1. get dishes from the database through your `service`
2. send them to the frontend
3. if something breaks, return an error text to the frontend instead of data

That is why we write `try/catch` inside the function.

`try` is the block where the main code lives.

`catch` is the block that will run if something inside `try` goes wrong.

The full code will look like this:

```js
// import code from your service
import * as dishService from '../services/dish/dish.service.js';

// your function
export async function list(req, res) {
  try {
    const dishes = await dishService.getDishes();

    res.json({ dishes });
  } catch (err) {
    res.status(500).json({ message: 'Something went wrong' });
  }
}
```

What does the function do?

1. When the request from the frontend reaches routes, Express calls your `list` function.
2. Inside `try`, you call `dishService.getDishes()`.
3. `getDishes` from `dish.service.js` is called.
4. Service `getDishes` calls `dishRepository.getDishes()` from `dish.repository.js`.
5. Repository `getDishes` goes to the database and returns the list of dishes.
6. The dishes return back up.
7. You save them into the `dishes` variable.
8. Then you send the response with `res.json({ dishes })`. This is a ready Express command for sending a JSON response to the frontend.
9. If something goes wrong somewhere, `catch` catches it and returns an error response. 500 is a backend error code.

Done!

## Testing the route

Now you need to check that your route really works.

First open a terminal, go to the `backend` folder, and run the backend with this command:

```bash
cd backend
npm run dev
```

Then just open in the browser:

```http
http://localhost:3000/api/v1/dishes
```

If everything is written correctly, you will see JSON with the list of dishes on the screen.

Example of what you will see:

```json
{
  "dishes": [
    {
      "id": 1,
      "name": "Sake Sashimi",
      "description": "8 slices salmon sashimi with wasabi, ginger, soy sauce and seaweed salad.",
      "price": 24,
      "is_available": true,
      "created_at": "2026-04-23T10:00:00.000Z"
    },
    {
      "id": 2,
      "name": "Moguro",
      "description": "6 slices tuna sashimi with wasabi, ginger and marinated cucumber.",
      "price": 24,
      "is_available": true,
      "created_at": "2026-04-23T10:00:00.000Z"
    },
    {
      "id": 3,
      "name": "Hamachi",
      "description": "8 slices yellowtail sashimi with ponzu, pickled daikon and herb salad.",
      "price": 26,
      "is_available": true,
      "created_at": "2026-04-23T10:00:00.000Z"
    }
  ]
}
```

What to check:

- the request does not crash
- the response has the `dishes` field
- `dishes` is an array
- inside the array, dishes have `id`, `name`, `description`, `price`, `is_available`, `created_at`

If that is true, then your route works.

## Next step: independent work with daily special

Create a repository, service, and controller for the request that should return the dish of the day.

```http
GET /api/v1/dishes/daily-special
```

Instructions:

1. Open:

- `backend/src/models/db/schemas/02_tables.sql`
- `backend/src/models/db/schemas/04_dummy_data.sql`

2. Look at the tables:

- `dishes`
- `daily_specials`

3. Open:

- `backend/src/routes.js`

4. Find the route:

```js
router.get('/dishes/daily-special', dishController.specials);
```

5. Go to the folder:

- `backend/src/models/db/repositories/dish/`

6. Create the file:

- `dailySpecial.repository.js`

7. In this file create the function:

- `getDailySpecial`

8. Inside this function write the SQL query:

```sql
SELECT
  dishes.id,
  dishes.name,
  dishes.description,
  dishes.price,
  dishes.is_available,
  dishes.created_at
FROM daily_specials
JOIN dishes
  ON dishes.id = daily_specials.dish_id
WHERE daily_specials.valid_on = CURDATE()
LIMIT 1
```

9. Open:

- `backend/src/services/dish/dish.service.js`

10. Add the function there:

- `getDailySpecial`

11. Inside this function call:

- `dailySpecialRepo.getDailySpecial()`

12. Return the result.

13. Open:

- `backend/src/controllers/dish.controller.js`

14. Find the line:

```js
export const specials = placeholder('dishes.specials');
```

15. Delete it.

16. In its place create the function:

- `specials`

17. Inside the function:

- call `dishService.getDailySpecial()`
- save the result into the `dish` variable
- return the response with `res.json({ dish })`
- in `catch`, return the error with `res.status(500).json(...)`


18. Run:

```bash
cd backend
npm run dev
```

19. Open in the browser:

```http
http://localhost:3000/api/v1/dishes/daily-special
```

20. Check that the response has the field:

- `dish`
