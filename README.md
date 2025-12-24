# hackernewsalerts-backend

The backend services for [hackernewsalerts.com](https://hackernewsalerts.com). A web application that sends email notifications when someone replies to one of your comments or posts on Hacker News.

[Link to the frontend](https://github.com/frostyalce000/hackernewsalerts-frontend).

## Features

- **Email Signup**: Users can sign up with their Hacker News username and email address
- **Email Verification**: Secure email verification system using signed tokens
- **Comment Replies Monitoring**: Automatically detects new replies to user's comments
- **Post Comments Monitoring**: Tracks new comments on user's posts (within the last 14 days)
- **Email Alerts**: Sends consolidated email notifications with all new activity
- **Unsubscribe**: Easy one-click unsubscribe functionality
- **Error Monitoring**: Admin notifications when processing errors occur

## Architecture

The project consists of 2 Python Django services that share the same PostgreSQL database:

### 1. REST API Service (Web)
- **Framework**: [Django Ninja](https://github.com/vitalik/django-ninja) for building REST APIs
- **Purpose**: Handles user signups, email verification, and unsubscribe requests
- **Server**: Gunicorn WSGI server
- **File**: [`alerts/views.py`](./alerts/views.py)

### 2. Scheduled Worker Service (Tasks)
- **Framework**: [Django Q2](https://django-q2.readthedocs.io/en/master/) for task scheduling
- **Purpose**: Periodically checks for new Hacker News activity and sends email alerts
- **File**: [`alerts/tasks.py`](./alerts/tasks.py)

Both services are deployed using [CapRover](https://caprover.com/) with separate Docker containers.

## Tech Stack

- **Python 3.12**
- **Django 5.0.6** - Web framework
- **Django Ninja 1.2.0** - REST API framework
- **Django Q2 1.6.2** - Task queue and scheduling
- **PostgreSQL** - Database (production) / SQLite (development)
- **AWS SES (boto3)** - Email delivery service
- **Gunicorn** - WSGI HTTP server
- **Docker** - Containerization
- **CapRover** - Deployment platform

### Key Dependencies

- `boto3` - AWS SDK for email sending via SES
- `beautifulsoup4` - HTML parsing for comment content
- `requests` - HTTP client for Hacker News RSS feeds
- `pydantic` - Data validation
- `django-cors-headers` - CORS handling
- `whitenoise` - Static file serving

## Project Structure

```
hackernewsalerts-backend/
├── alerts/                    # Main Django app
│   ├── models.py             # User model
│   ├── views.py              # REST API endpoints
│   ├── tasks.py              # Scheduled task for checking alerts
│   ├── hn.py                 # Hacker News API integration
│   ├── mail.py               # Email sending via AWS SES
│   ├── utils.py              # Utility functions
│   ├── urls.py               # URL routing
│   ├── admin.py              # Django admin configuration
│   └── migrations/           # Database migrations
├── socialalerts/             # Django project settings
│   ├── settings.py           # Django configuration
│   ├── urls.py               # Root URL configuration
│   ├── wsgi.py               # WSGI application
│   └── asgi.py               # ASGI application
├── Dockerfile-web            # Docker image for API service
├── Dockerfile-tasks          # Docker image for worker service
├── captain-definition-web.json    # CapRover config for web
├── captain-definition-tasks.json  # CapRover config for tasks
├── requirements.txt          # Python dependencies
├── Makefile                  # Common commands
├── run-web.sh               # Web service startup script
└── manage.py                # Django management script
```

## Setup & Installation

### Prerequisites

- Python 3.12+
- PostgreSQL (for production) or SQLite (for development)
- AWS account with SES configured (for email sending)
- Docker (optional, for containerized deployment)

### Local Development Setup

1. **Clone the repository**
   ```bash
   git clone https://github.com/frostyalce000/hackernewsalerts-backend.git
   cd hackernewsalerts-backend
   ```

2. **Create a virtual environment**
   ```bash
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   ```

3. **Install dependencies**
   ```bash
   pip install -r requirements.txt
   ```

4. **Create a `.env` file** (see [Configuration](#configuration) section)
   ```bash
   cp .env.example .env  # If you have an example file
   # Or create .env manually with required variables
   ```

5. **Run database migrations**
   ```bash
   make migrate
   # or
   python manage.py migrate
   ```

6. **Create a superuser** (optional, for admin access)
   ```bash
   make create-superuser
   # or
   python manage.py createsuperuser
   ```

### Running the Services

#### Run the Web API Service
```bash
make run-web
# or
python manage.py runserver
```

The API will be available at `http://localhost:8000`

#### Run the Task Worker Service
```bash
make run-tasks
# or
python manage.py qcluster
```

**Note**: The scheduled task (`check_for_alerts`) needs to be configured in Django Q2's admin interface or via management command. The task should be scheduled to run periodically (e.g., every hour).

## Configuration

### Environment Variables

The application uses environment variables for configuration. In local development, these are loaded from a `.env` file using `python-dotenv`.

#### Required Variables

- `SECRET_KEY` - Django secret key (required in production)
- `DB_NAME` - PostgreSQL database name (production)
- `DB_USER` - PostgreSQL database user (production)
- `DB_PASSWORD` - PostgreSQL database password (production)
- `DB_HOST` - PostgreSQL database host (production)
- `DB_PORT` - PostgreSQL database port (production)
- `UI_URL` - Frontend URL for email verification links (e.g., `https://hackernewsalerts.com`)
- `API_URL` - Backend API URL for unsubscribe links (e.g., `https://api.hackernewsalerts.com`)

#### AWS SES Configuration

The application uses AWS SES for sending emails. Configure AWS credentials using one of these methods:

1. **AWS Credentials File** (`~/.aws/credentials`)
   ```ini
   [default]
   aws_access_key_id = YOUR_ACCESS_KEY
   aws_secret_access_key = YOUR_SECRET_KEY
   ```

2. **Environment Variables**
   ```bash
   export AWS_ACCESS_KEY_ID=your_access_key
   export AWS_SECRET_ACCESS_KEY=your_secret_key
   export AWS_DEFAULT_REGION=eu-west-2
   ```

3. **IAM Role** (if running on AWS infrastructure)

**Note**: The sender email `alerts@hackernewsalerts.com` must be verified in AWS SES.

### Django Settings

Key settings in `socialalerts/settings.py`:

- **Database**: Uses PostgreSQL in production, SQLite in development (when `SECRET_KEY` is not set)
- **CORS**: Configured for frontend domains
- **Django Q2**: Configured with ORM backend, 1 worker, 12000s timeout
- **Logging**: Console logging with verbose format

## API Endpoints

All API endpoints are prefixed with `/api/` and use Django Ninja.

### POST `/api/signup`
Create a new alert subscription.

**Request Body:**
```json
{
  "hn_username": "username",
  "email": "user@example.com"
}
```

**Response:**
- `201 Created` - User created, verification email sent
- `400 Bad Request` - Username already exists

### POST `/api/verify/{code}`
Verify email address using verification code from email.

**Response:**
- `200 OK` - Email verified successfully
- `400 Bad Request` - Invalid verification code
- `404 Not Found` - User not found

### GET `/api/unsubscribe/?token={token}`
Preview unsubscribe page (confirmation form).

**Response:**
- HTML page with unsubscribe confirmation form
- `400 Bad Request` - Invalid token

### POST `/api/unsubscribe/confirm/?token={token}`
Confirm and process unsubscribe request.

**Response:**
- `200 OK` - Successfully unsubscribed
- `400 Bad Request` - Invalid token
- `404 Not Found` - User not found

## Database Models

### User Model

Located in `alerts/models.py`:

```python
class User(models.Model):
    hn_username = models.CharField(max_length=100, db_index=True, unique=True)
    email = models.EmailField(db_index=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    last_checked = models.DateTimeField(default=timezone.now)
    is_verified = models.BooleanField(default=False)
```

**Fields:**
- `hn_username` - Hacker News username (unique, indexed)
- `email` - User's email address (indexed)
- `created_at` - Account creation timestamp
- `updated_at` - Last update timestamp
- `last_checked` - Last time alerts were checked for this user
- `is_verified` - Whether the email has been verified

## How It Works

### Signup Flow

1. User submits signup form with HN username and email
2. System creates a `User` record with `is_verified=False`
3. Verification email is sent with a signed token
4. User clicks verification link in email
5. System verifies token and sets `is_verified=True`
6. User is now eligible to receive alerts

### Alert Checking Flow

The scheduled task (`check_for_alerts`) runs periodically:

1. Fetches all verified users
2. For each user:
   - Calls `get_new_comment_replies()` to find replies to user's comments
   - Calls `get_new_post_comments()` to find comments on user's posts (last 14 days)
   - Filters out comments/replies by the user themselves
   - Filters by `last_checked` timestamp to get only new activity
3. If new activity found:
   - Formats email content with all new comments/replies
   - Adds unsubscribe link
   - Sends email via AWS SES
4. Updates `user.last_checked` timestamp
5. If errors occur, sends alert email to admin

### Hacker News Integration

The system uses [hnrss.org](https://hnrss.org/) RSS feeds:

- **Comment Replies**: `https://hnrss.org/replies.jsonfeed?id={username}`
- **User Posts**: `https://hnrss.org/submitted.jsonfeed?id={username}`
- **Post Comments**: `https://hnrss.org/item.jsonfeed?id={post_id}`

Only posts from the last 14 days are checked for comments (posts older than 14 days are typically closed for discussion).

## Development

### Common Commands

See the [Makefile](./Makefile) for all available commands:

```bash
# Run web service
make run-web

# Run task worker
make run-tasks

# Database migrations
make makemigrations
make migrate

# Create superuser
make create-superuser

# Collect static files
make collect-static

# Run tests
make test

# Docker commands
make docker-build-web
make docker-run-web

# Deployment (requires environment variables)
make deploy-web
make deploy-tasks
```

### Code Structure

- **`alerts/hn.py`**: Functions to fetch data from Hacker News RSS feeds
- **`alerts/mail.py`**: AWS SES email sending wrapper
- **`alerts/utils.py`**: Utility functions (date formatting, HTML parsing, token signing)
- **`alerts/tasks.py`**: Main task function that processes all users
- **`alerts/views.py`**: REST API endpoint handlers

## Testing

Run tests with:
```bash
make test
# or
python manage.py test
```

### Test Files

- `alerts/tests.py` - General tests
- `alerts/tests_hn.py` - Hacker News integration tests
- `alerts/tests_tasks.py` - Task processing tests

**Note**: Some tests require environment variables (e.g., `TEST_RUN_TASK=1`, `TEST_HN_USERNAME`) to run integration tests against real Hacker News data.

## Deployment

### Docker

The project includes two Dockerfiles:

- **`Dockerfile-web`**: For the REST API service
  - Runs migrations on startup
  - Starts Gunicorn server on port 8000

- **`Dockerfile-tasks`**: For the scheduled worker service
  - Runs Django Q2 cluster (`qcluster`)

### CapRover Deployment

The project is configured for CapRover deployment with two separate apps:

1. **Web Service**: Deployed using `captain-definition-web.json`
2. **Tasks Service**: Deployed using `captain-definition-tasks.json`

Deploy using:
```bash
make deploy-web    # Requires CAPROVER_APP_WEB, CAPROVER_NODE, BRANCH
make deploy-tasks  # Requires CAPROVER_APP_TASKS, CAPROVER_NODE, BRANCH
```

### Production Checklist

- [ ] Set `SECRET_KEY` environment variable
- [ ] Configure PostgreSQL database connection
- [ ] Set up AWS SES and verify sender email
- [ ] Configure `UI_URL` and `API_URL` environment variables
- [ ] Set up Django Q2 scheduled task for `check_for_alerts`
- [ ] Configure CORS allowed origins
- [ ] Set up proper logging and monitoring
- [ ] Create superuser for admin access

## Notes

### Known Issues

- The scheduled worker sends email alerts to the admin user every time there's a failure. Sometimes these alerts occur due to missing data from the Hacker News RSS feeds, but the data usually becomes available after a couple of hours.

### Security Considerations

- Email verification uses Django's signing framework with secure tokens
- Unsubscribe tokens are signed with a separate salt
- CSRF protection is enabled for POST endpoints
- CORS is configured to restrict origins
- Database credentials should never be committed to version control

### Performance

- User queries are indexed on `hn_username` and `email`
- Only posts from the last 14 days are checked for comments
- The task processes users sequentially; consider parallelization for large user bases

## License

See [LICENSE](./LICENSE) file for details.

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.
