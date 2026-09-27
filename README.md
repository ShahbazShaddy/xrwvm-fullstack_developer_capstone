# Cars Dealership — Full-Stack Capstone

A national car dealership web application built for the IBM Full-Stack Developer
Capstone. Visitors can browse dealerships across the US, filter them by state and
read customer reviews. Every review is tagged with a sentiment (positive, neutral
or negative). Registered users can log in and post their own reviews of a dealer.

## Features

- Home, About Us and Contact Us pages
- User registration, login and logout (Django authentication)
- List all dealerships, or filter them by state
- Dealer details page with its reviews and a sentiment icon for each review
- Post a review (logged-in users only), choosing from the car makes and models
  managed in the Django admin

## Architecture

```
                     ┌──────────────────────────────────────────┐
  Browser  ────────▶ │ Django app (port 8000)                   │
  (React SPA +       │  - serves the React build + static pages │
   static pages)     │  - auth, CarMake/CarModel (SQLite)       │
                     │  - proxy views in djangoapp/restapis.py  │
                     └───────────┬──────────────────┬───────────┘
                                 │                  │
                                 ▼                  ▼
          ┌────────────────────────────┐   ┌──────────────────────────┐
          │ Node/Express API (3030)    │   │ Flask sentiment analyzer │
          │ dealers + reviews          │   │ NLTK VADER (5050)        │
          │        │                   │   │ /analyze/<text>          │
          │        ▼                   │   └──────────────────────────┘
          │ MongoDB (27017)            │
          └────────────────────────────┘
```

| Component | Location | Tech |
|-----------|----------|------|
| Web app & auth | `server/djangoproj`, `server/djangoapp` | Django, SQLite |
| Frontend | `server/frontend` | React 18, React Router, Bootstrap |
| Dealers & reviews API | `server/database` | Node.js, Express, Mongoose, MongoDB |
| Sentiment analysis | `server/djangoapp/microservices` | Flask, NLTK VADER |
| CI | `.github/workflows/main.yml` | flake8, jshint |
| Deployment | `server/Dockerfile`, `server/deployment.yaml` | Docker, Kubernetes |

### Django endpoints (`/djangoapp/...`)

| Method | Path | Description |
|--------|------|-------------|
| POST | `login` | body `{"userName", "password"}` → `{"userName", "status": "Authenticated"}` |
| GET | `logout` | → `{"userName": ""}` |
| POST | `register` | body `{"userName", "password", "firstName", "lastName", "email"}` |
| GET | `get_cars` | car makes and models (seeds data on first call) |
| GET | `get_dealers`, `get_dealers/<state>` | all dealers / dealers in a state |
| GET | `dealer/<id>` | dealer details |
| GET | `reviews/dealer/<id>` | dealer reviews with sentiment |
| POST | `add_review` | post a review (logged-in users only) |

### Node API (port 3030)

`/fetchReviews`, `/fetchReviews/dealer/:id`, `/fetchDealers`,
`/fetchDealers/:state`, `/fetchDealer/:id`, `POST /insert_review`

## Running locally

Prerequisites: Python 3.9+, Node.js 18+, Docker (for MongoDB and the Node API).

1. **MongoDB and the Node API** (port 3030)

   ```bash
   cd server/database
   docker build . -t nodeapp
   docker compose up -d
   ```

2. **Sentiment analyzer** (port 5050)

   ```bash
   cd server/djangoapp/microservices
   pip install -r requirements.txt
   python app.py
   ```

3. **Configure Django**: create `server/djangoapp/.env`:

   ```
   backend_url=http://localhost:3030
   sentiment_analyzer_url=http://localhost:5050/
   ```

4. **Build the React frontend**

   ```bash
   cd server/frontend
   npm install
   npm run build
   ```

5. **Run Django** (port 8000)

   ```bash
   cd server
   python -m venv venv && source venv/bin/activate   # Windows: venv\Scripts\activate
   pip install -r requirements.txt
   python manage.py makemigrations
   python manage.py migrate
   python manage.py createsuperuser
   python manage.py runserver
   ```

Open http://localhost:8000. The Django admin is at http://localhost:8000/admin.

## Deployment

`server/Dockerfile` and `server/entrypoint.sh` build a container image of the
Django app. `server/deployment.yaml` deploys it to Kubernetes as the
`dealership` deployment. Replace `<namespace>` in `deployment.yaml` with your
IBM Cloud Container Registry namespace before you apply it.

## License

Apache 2.0. See [LICENSE](LICENSE).
