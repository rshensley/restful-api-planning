# Solar System Object Explorer

**INFO U762 — Interoperability | Module 4: RESTful API Planning with Fetch GET**

## 1. API Selection

The **Solar System Object Explorer** is a browser-based application that retrieves information about planets, moons, and other Solar System bodies. Users will search for an object, review its physical and orbital properties, and can save it to a personal observing list with notes.

The application will consume the **Solar System OpenData API**, which is included in the [Public APIs GitHub directory](https://github.com/public-apis/public-apis). Its documented endpoint, `GET https://api.le-systeme-solaire.net/rest/bodies/{id}`, returns information about a specific body as JSON. Available data include classification, parent planet, radius, mass, orbital parameters, and discovery information.

**Titan**, Saturn's largest moon, will be the primary example. A successful test of `GET /rest/bodies/titan` returned fields including `englishName: "Titan"`, `bodyType: "Moon"`, `meanRadius: 2575.0`, `sideralOrbit: 15.95`, and `aroundPlanet.planet: "saturne"`. A request for a nonexistent object returned HTTP `404` with the plain-text response `Not found (entity)`.

The API requires a free **Bearer token**. Although the Public APIs directory lists it without authentication, the provider's current documentation and the successful test confirm that the token is required. The proposed Flask backend will supply the token when requesting external data and it will not be included in browser-side JavaScript.

## 2. Endpoint Mapping

The external service supplies read-only object information. The application will provide its own REST endpoints for searching and managing saved observing-list entries.

| Method | Endpoint | Purpose | Typical response |
|---|---|---|---|
| `GET` | `https://api.le-systeme-solaire.net/rest/bodies/{id}` | Retrieve body data from Solar System OpenData; requires Bearer authentication | `200` with JSON; `404` for an unknown body |
| `GET` | `/api/bodies/{id}` | Flask retrieves an external body and returns a JSON response to the browser | `200` with JSON; `404` if not found |
| `GET` | `/api/observing-list` | Read saved observing-list entries | `200` with JSON array |
| `POST` | `/api/observing-list` | Create a saved entry with optional notes | `201 Created` |
| `PATCH` | `/api/observing-list/{id}` | Update the notes on an existing entry | `200 OK` |
| `DELETE` | `/api/observing-list/{id}` | Delete a saved entry | `204 No Content` |

The Flask endpoints cover **Create, Read, Update, and Delete (CRUD)**.

This design follows REST principles:

- **Resource identification:** URIs identify individual bodies (`/api/bodies/titan`) and saved records (`/api/observing-list/{id}`).
- **Standard HTTP methods:** `GET` retrieves representations without changing resources; `POST` creates an entry; `PATCH` updates its notes; and `DELETE` removes it.
- **Stateless requests:** Each HTTP request contains the information needed to process it. The backend retrieves saved data from SQLite instead of relying on a previous request or server-side session state.
- **Resource representations:** JSON is used to transfer body information and observing-list records between the browser and Flask. HTTP status codes distinguish successess from errors.

## 3. Data Flow Diagram (Textual)

1. **User search:** The user enters a Solar System body identifier, such as `titan`, into a search form.
2. **Client-side validation:** JavaScript trims the input, rejects an empty value, and URL-encodes a valid identifier.
3. **Asynchronous request:** JavaScript displays a loading indicator and uses `fetch()` with an `Accept: application/json` header to send a `GET` request to the application's `/api/bodies/titan` endpoint. The interface remains responsive while the request is pending.
4. **External API request:** Flask uses Python's `requests` library to send an HTTPS `GET` request to `https://api.le-systeme-solaire.net/rest/bodies/titan`, including the Bearer token from a server-side environment variable.
5. **Response processing:** Flask checks the external HTTP status and parses successful JSON data. It returns a consistent JSON response to the browser. For an unknown object, it can translate the external API's plain-text `404` into a JSON error response; it also handles authentication failures, timeouts, and other upstream errors.
6. **Dynamic rendering:** JavaScript checks `response.ok`, parses the returned JSON, and builds a results table with labeled rows for fields such as `englishName`, `bodyType`, `meanRadius`, `sideralOrbit`, and the parent planet. The interface displays only available, meaningful values.
7. **Conditional styling:** JavaScript selects CSS classes based on returned data; for example, `body-moon` for an object whose `bodyType` is `"Moon"` and `body-planet` for a planet. Separate classes distinguish loading, success, errors, and unavailable values. Missing or questionable measurements display as **“Not available”** rather than misleading zeros.
8. **Optional persistence:** If the user saves Titan with notes, JavaScript sends a JSON `POST` request with `Content-Type: application/json` to `/api/observing-list`. Flask validates the payload, stores it in SQLite, and returns confirmation. Saved entries can subsequently be retrieved, edited, or deleted through the corresponding endpoints.

This flow illustrates interoperability between the external astronomical-data provider, the Python backend, and the JavaScript frontend. Some upstream values may need transformation: for example, Titan's parent planet is represented as `"saturne"`, which the interface can display as **Saturn**. The tested response also contains zero-valued fields such as `gravity` and `avgTemp`; these should be checked for completeness before being displayed.

## 4. Form Handling

**Object search form:** The user enters a body identifier such as `titan` or `mars`. JavaScript trims the input, requires **1–50 characters**, and permits letters, numbers, spaces, and hyphens (subject to confirmation against the provider's supported identifiers). It URL-encodes a valid identifier before sending `GET /api/bodies/{id}` with `Accept: application/json`. Flask independently validates the identifier before contacting the external API. A nonexistent object produces a clear “Object not found” message.

**Observing-list form:** After viewing a body, the user can save its body identifier and optional observing notes. The body identifier is required; notes are optional and limited to **500 characters**. JavaScript validates these rules and submits the JSON payload to `POST /api/observing-list` with `Content-Type: application/json`. For example:

```json
{
  "body_id": "titan",
  "notes": "Look for Titan near Saturn during the next clear night."
}
```
The Flask backend validates the submitted JSON before creating a SQLite record. To edit a saved entry, JavaScript submits a JSON `PATCH /api/observing-list/{id}` request with updated notes (again limited to 500 characters). The `{id}` must identify an existing saved record; an unknown record returns `404`. Users can remove a record with `DELETE /api/observing-list/{id}`. Success and failure messages will be displayed in the interface. Validation on both the frontend and backend helps prevent incomplete or malformed data from being stored.

## 5. Technology Stack

| Technology | Planned use |
|---|---|
| **HTML5 and CSS3** | Search form, results display, observing-list interface, loading and error styles |
| **JavaScript (Fetch API)** | Form handling, asynchronous requests to Flask, JSON parsing, dynamic rendering |
| **Python and Flask** | Local REST endpoints, validation, and coordination with the external API |
| **Python Requests** | Authenticated HTTP calls from Flask to Solar System OpenData |
| **SQLite** | Persistent storage for saved bodies and observing notes |
| **Git and GitHub** | Version control and documentation |
| **Browser Developer Tools, curl, pytest** | Inspect network requests, test endpoints and errors, and verify backend behavior |

The browser will communicate with Flask rather than exposing the provider's Bearer token in JavaScript. Flask will read the token from an environment variable.

## 6. Implementation Steps

1. **Confirm the API and data format.** Review the provider documentation and repeat the successful Titan request. Record the expected JSON fields and HTTP status codes, including the observed `404` for an unknown object.
2. **Set up the project.** Create a Flask project, Python virtual environment, dependency list, Git repository, and local configuration for the API token.
3. **Implement the object lookup.** Build `GET /api/bodies/{id}` using Python Requests. Add a timeout, handle external HTTP errors, and return consistent JSON to the frontend.
4. **Build the browser interface.** Create the search form and a results table with HTML and CSS. Add asynchronous JavaScript Fetch, request headers, the specified input and notes-length validation, loading/error states, and conditional styling based on object classification.
5. **Implement observing-list CRUD.** Add SQLite storage and Flask endpoints to create, read, update, and delete saved entries. Validate JSON on the server.
6. **Test integration and failures.** Verify searches for Titan and other supported bodies, empty and extra long identifiers, notes over 500 characters, a `404` response, unavailable fields, authentication errors, network timeouts, and all four observing list operations (including PATCH and DELETE of nonexistent IDs). Confirm SQLite records persist after restarting Flask.
7. **Document and prepare for deployment.** Write setup instructions, document environment variables, keep credentials out of source control, and configure HTTPS and a production server with Flask debugging disabled if the application is deployed.

## References

- Public APIs. [Public APIs GitHub directory](https://github.com/public-apis/public-apis).
- Solar System OpenData. [Official API website and documentation](https://api.le-systeme-solaire.net/en/).
