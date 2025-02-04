---
{"dg-publish":true,"dg-path":"Programming Tutorials/Literature Note - Pymongo Tutorial using an Atlas Cluster.md","permalink":"/programming-tutorials/literature-note-pymongo-tutorial-using-an-atlas-cluster/","tags":["mongodb","fastapi","Tutorial"]}
---

# What is a CRUD application?

CRUD, in different terms:

| CRUD   | SQL    | HTTPS                           |
| ------ | ------ | ------------------------------- |
| Create | INSERT | POST                            |
| Read   | SELECT | GET                             |
| Update | UPDATE | PUT to replace, PATCH to modify |
| Delete | DELETE | DELETE                          |

### In Concept

Data can be stored.

- Such storage must allow its data to be _readable_ and _updatable_.
- Before such actions can be taken, the storage location must _allocate_ and _initialize_ the content.
- At some later point, the storage can be _deleted_ and _deallocated_.

These constitute the basic operations of storage management: **CRUD**.

## Learning MongoDB with pymongo

This project focuses on learning by doing. We will follow the tutorial found on MongoDB's website. For now, we will use the cloud solution, MongoDB Atlas, but in future projects, we will use Docker and self-hosting solutions.

> [!note] Sources
> https://www.mongodb.com/resources/languages/pymongo-tutorial

## Requirements

- Python
- FastAPI
- python-dotenv
- pydantic

## Project Setup

A conda virtual environment was created with the required packages, and a MongoDB account was created to use Atlas. For future projects, I will try to use native MongoDB solutions.

Using `python-dotenv`, we establish our variables to connect to the Atlas Cluster:

- `ATLAS_URI` -> The connection string used by `pymongo` to connect to the cluster.
- `DB_NAME` -> The name of the database where we will create the CRUD actions.

We also create the following files: `models.py`, `routes.py`, and `main.py`

## `models.py`

Using `pydantic`'s `BaseModel` and the (deprecated) `class Config`, we create data classes for data validation. The `Config` class is used to determine some behavior of the model, and `fastapi` then leverages the model structure for documentation, leveraging `json_schema_extra` to add details like examples.

>[!note]- config
> While this example uses a deprecated configuration class, it is important to note that it can be updated by replacing the class with the following:
>```python
>	model_config = ConfigDict(
>        populate_by_name=True,
>        json_schema_extra={
>            "example": {
>                "_id": "066de609-b04a-4b30-b46c-32537c7f1f6e",
>                "title": "Don Quixote",
>                "author": "Miguel de Cervantes",
>                "synopsis": "..."
>            }
>        }
>    )
>```

We create two models:

1. `Book`

- This model validates the book data when creating a new book, taking in the title, author, and synopsis.

2. `BookUpdate`

- Used to validate data before updating a book in the database.

## `routes.py`

This file creates all the CRUD functions needed for the application. FastAPI handles URL routing, request processing, and automatic interactive API documentation (docs), while Pydantic, through the imported models, handles data validation.

### CREATE

`@router.post("/", response_description = ..., status_code = status.HTTP_201_CREATED, response_model = Book)`

Here, we define our POST request at the root path. If the function below is executed correctly, the API returns a 201 code.

```python
def create_book(request: Request, book: Book = Body(...)):
    book = jsonable_encoder(book)
    new_book = request.app.database["books"].insert_one(book)
    created_book = request.app.database["books"].find_one(
        {"_id": new_book.inserted_id}
    )
    return created_book
```

The function `create_book` creates a new `Book` entry in the database. FastAPI automatically parses the incoming request body according to the `Book` model and Pydantic validates the data. The validated `Book` object is converted to a JSON-serializable dictionary, and then inserted into the "books" collection in the database using pymongo's `insert_one()` method.

> [!note] `new_book = ?` 
> Poor naming: `new_book` does not contain the book we inserted into the database; instead, it contains the return type `InsertOneResult` from the `insert_one()` function. This type of object has two attributes:
>
> - `inserted_id`: The inserted document's ID.
> - `acknowledged`: The response to the write operation.

Finally, we query to find the inserted book and return it as to check for a successful insertion.

### READ

`@router.get("/", response_description="...", response_model=List[Book])`

Here, we tell FastAPI that calling the function will handle GET requests to the root path, and we specify the type of object we will return in the body of the request.

```python
def list_books(request: Request):
    books = list(request.app.database["books"].find(limit=100))
    return books
```

Our `list_books` function, which lists the first 100 books in our database. But what if we want a specific book?

```python
@router.get("/{id}", response_description="Get a single book by id", response_model=Book)
def find_book(id: str, request: Request):
    if (book := request.app.database["books"].find_one({"_id": id})) is not None:
        return book
    raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail=f"Book with ID {id} not found")
```


Let's break it down line by line. First, the route definition captures the `id` from the URL path. Then, the `find_book` function takes the `id` as a parameter. The code uses the walrus operator to attempt to retrieve the book from the database using pymongo's `find_one` function. If a book with the given `id` is found, it is assigned to the `book` variable and returned. If no book is found, an `HTTPException` with a 404 Not Found status is raised.

### UPDATE

`@router.put("/{id}", response_description="...", response_model=Book)`

As before, we specify that the following function will make a `PUT` request to the indicated path and assign the value to `id`.


```python
def update_book(id: str, request: Request, book: BookUpdate = Body(...)):
    book = {k: v for k, v in book.dict().items() if v is not None}
    if len(book) >= 1:
        update_result = request.app.database["books"].update_one(
            {"_id": id}, {"$set": book}
        )

        if update_result.modified_count == 0:
            raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail=f"Book with ID {id} not found")

    if (
        existing_book := request.app.database["books"].find_one({"_id": id})
    ) is not None:
        return existing_book

    raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail=f"Book with ID {id} not found")
```

The `update_book` function uses `update_one` with the `$set` operator (implicitly) to update a book. It first filters the input `BookUpdate` model, removing `None` values to allow for partial updates. It then performs the update based on the provided `id`. After the update attempt, it retrieves the updated book.

> [!note] `$set` 
>- This operator will only update the given fields of the book passed into it. That is why we filtered `None` values before.

The final `raise` handles the case where no book with the given `id` was found, either initially or if it was somehow deleted between the update and the subsequent retrieval.

### DELETE

```python
@router.delete("/{id}", response_description="Delete a book")
def delete_book(id: str, request: Request, response: Response):
    delete_result = request.app.database["books"].delete_one({"_id": id})

    if delete_result.deleted_count == 1:
        response.status_code = status.HTTP_204_NO_CONTENT
        return response

    raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail=f"Book with ID {id} not found")
```

The `delete_book` function uses the `Response` class to explicitly set the HTTP status code. It uses the `delete_result.deleted_count` to verify successful deletion and, if successful, sets the status code to 204 No Content, which is common for successful DELETE operations.

## `main.py`

Here, we put everything together.

`from routes import router as book_router`

We import the router we created in `routes.py`, which contains all of the CRUD functions we created earlier.

`config = dotenv_values(".env")`

We read the environmental variables that we need to connect to the MongoDB cluster.

```python
#Deprecated on_event
@app.on_event("startup")
def startup_db_client():
    app.mongodb_client = MongoClient(config["ATLAS_URI"])
    app.database = app.mongodb_client[config["DB_NAME"]]
    print("Connected to the MongoDB database!")
```

Our function for initializing our site. We start by creating a `MongoClient` (connecting to the cluster). Then, we specify database we are going to connect to, and finally, we indicate that we are connected.

> [!info] `app.mongodb_client` 
>- When we store the `MongoClient` instance as part of the `app` object (FastAPI), we make it globally available. This allows us to put the CRUD functions in the `router.py` file, where we access the database using the `request.app` object.


```python
#Deprecated on_event
@app.on_event("shutdown")
def shutdown_db_client():
    app.mongodb_client.close()
```

Simple: close the client connection when shutting down the application.

```python
app.include_router(book_router, tags=["books"], prefix="/book")
```

Since we created the routing operations in a different file, we need to indicate as such with the `include_router` function. For better readability of our API, we tag it under `books`, and we add a prefix to the routing operations such that if, in the `routes.py` file, they are on path `/`, they will instead be on path `/book/`.