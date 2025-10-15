# Documentation for `auth.py`

This document provides a comprehensive technical overview and line-by-line documentation for the provided Python script, which constitutes an authentication module for an R2 API.

---

## R2 API Authentication Module Documentation

### Overview

This Python module, `auth_router.py`, serves as the core authentication component for the R2 API, built using FastAPI. It provides a robust set of functionalities for user management, including registration, login, token-based authentication (using JWT), password management (hashing and reset flows), and team listing. The module leverages `bcrypt` for secure password hashing and `PyJWT` for JSON Web Token generation and verification. It integrates with an `AuthAdapter` for database interactions and uses `FastAPI`'s `APIRouter` to define API endpoints. Background tasks are utilized for non-blocking email notifications.

**Key Features:**
*   **User Registration:** Allows new users to sign up, hashing their passwords securely.
*   **User Login:** Authenticates users and issues both access and refresh JWT tokens.
*   **Token Management:** Provides functions to create, refresh, and verify JWT tokens.
*   **Password Reset:** Implements a flow for users to request and perform password resets.
*   **Terms Agreement:** An endpoint to record a user's agreement to terms and conditions.
*   **Team Listing:** Retrieves a list of available teams, likely for registration purposes.
*   **Secure Practices:** Employs `bcrypt` for password hashing and `PyJWT` for secure token handling.
*   **Asynchronous Operations:** Uses FastAPI's `BackgroundTasks` for sending emails to avoid blocking API responses.

### Module Imports

The script begins by importing necessary libraries and modules:

```python
import jwt # Used for encoding, decoding, and verifying JSON Web Tokens (JWTs).
import bcrypt # Cryptographic library for hashing passwords securely.
import logging # Standard Python library for logging events and debugging information.
from datetime import datetime, timedelta # Used for handling dates and times, specifically for token expiration.
from fastapi import APIRouter, HTTPException, Depends, BackgroundTasks # FastAPI core components:
# APIRouter: To create a modular router for API endpoints.
# HTTPException: To raise standard HTTP errors.
# Depends: To manage dependencies, e.g., database adapters or token verification.
# BackgroundTasks: To run tasks in the background without blocking the main response.
from app.utils.error_handler import handle_exception # Custom utility to centralize error handling logic.
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm # FastAPI security utilities:
# OAuth2PasswordBearer: Implements OAuth2 password bearer flow for token authentication.
# OAuth2PasswordRequestForm: A Pydantic model for parsing username/password from form data.
from decouple import config # Library for managing environment variables (e.g., from .env files).
from app.models.auth_models import Token, UserRegister, TeamListModel # Pydantic models for request/response data validation:
# Token: Likely defines the structure for access and refresh tokens.
# UserRegister: Defines the data structure for user registration requests.
# TeamListModel: Defines the data structure for listing teams.
from app.db.auth_adapter import AuthAdapter # Custom adapter class for database interactions related to authentication.
from app.db.dependencies import get_auth_adapter # Dependency injector function to provide an AuthAdapter instance.
from app.services.password_generator import generate_secure_password # Service to generate cryptographically secure random passwords.
from app.services.mail_service import college_onboard_request_mail_to_user, college_onboard_request_mail_toadmin, registration_request_received_email, registration_request_received_email_to_admin, password_reset_request_mail # Email service functions for sending various notification emails.
```

### Configuration Variables

The script defines several configuration variables, primarily loaded from environment variables using `decouple.config` for secure and flexible deployment. Default values are provided for development convenience.

```python
JWT_SECRET = config("SECRET_KEY", default="your_secret_key")
# The secret key used for encoding and decoding JWTs. It MUST be kept confidential.
# Default: "your_secret_key" (should be changed to a strong, random value in production).

JWT_ALGORITHM = config("JWT_ALGORITHM", default="HS256")
# The cryptographic algorithm used for signing JWTs.
# Default: "HS256" (HMAC-SHA256).

ACCESS_TOKEN_EXPIRE_MINUTES = int(config("ACCESS_TOKEN_EXPIRE_MINUTES", default=1440))
# The expiration time for access tokens, in minutes.
# Default: 1440 minutes (24 hours).

REFRESH_TOKEN_EXPIRE_DAYS = int(config("REFRESH_TOKEN_EXPIRE_DAYS", default=7))
# The expiration time for refresh tokens, in days. Refresh tokens typically have a longer lifespan.
# Default: 7 days.

RESET_PASSWORD_URL = config("RESET_PASSWORD_URL", default="http://localhost:3000/reset-password")
# The base URL for the password reset page, used in password reset emails.
# Default: "http://localhost:3000/reset-password".
```

### Global Initialization

```python
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/login")
# Initializes an OAuth2PasswordBearer instance.
# tokenUrl="/auth/login" specifies the URL where clients can obtain a token (i.e., the login endpoint).
# This scheme is used as a dependency to extract the token from the Authorization header.

logging.basicConfig(level=logging.INFO)
# Configures the basic logging system.
# level=logging.INFO sets the minimum severity level for messages to be processed to INFO.
# This means INFO, WARNING, ERROR, and CRITICAL messages will be logged.

logger = logging.getLogger(__name__)
# Creates a logger instance for the current module, typically named after the module file.
# This allows for more granular control over logging from this specific module.

router = APIRouter()
# Initializes a FastAPI APIRouter.
# This router will group all the API endpoints defined in this file.
# It can then be included into the main FastAPI application.
```

### Core JWT Utility Functions

#### `create_jwt_token`

This function generates a standard JWT access token with an optional custom expiration time.

```python
def create_jwt_token(data: dict, expires_delta: timedelta = None) -> str:
    """
    Generate a JWT access token.
    
    Args:
        data (dict): Payload data to encode in the token. This dictionary will be
                     included as the claims in the JWT.
        expires_delta (timedelta, optional): Custom expiration time for the token.
                                             If not provided, the default `ACCESS_TOKEN_EXPIRE_MINUTES`
                                             from the configuration will be used.
        
    Returns:
        str: The encoded JWT token string.
    """
    to_encode = data.copy()
    # Creates a shallow copy of the input data dictionary to avoid modifying the original.

    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES))
    # Calculates the expiration timestamp for the token.
    # It takes the current UTC time and adds either the custom `expires_delta`
    # or the default `ACCESS_TOKEN_EXPIRE_MINUTES` duration.

    to_encode.update({"exp": expire})
    # Adds the calculated expiration timestamp (`exp`) to the token payload.
    # The 'exp' claim (expiration time) is a standard JWT claim.

    return jwt.encode(to_encode, JWT_SECRET, algorithm=JWT_ALGORITHM)
    # Encodes the payload (`to_encode`) into a JWT string using the global `JWT_SECRET`
    # and `JWT_ALGORITHM`.
```

#### `create_refresh_token`

This function generates a JWT refresh token, which typically has a longer expiration time than access tokens.

```python
def create_refresh_token(data: dict) -> str:
    """
    Generate a JWT refresh token with extended expiration.
    
    Args:
        data (dict): Payload data to encode in the token.
        
    Returns:
        str: The encoded refresh token string.
    """
    expire = datetime.utcnow() + timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS)
    # Calculates the expiration timestamp for the refresh token.
    # It adds the `REFRESH_TOKEN_EXPIRE_DAYS` duration to the current UTC time.

    data.update({"exp": expire})
    # Adds the calculated expiration timestamp (`exp`) to the token payload.
    # Note: This function directly modifies the input `data` dictionary, unlike `create_jwt_token`.

    return jwt.encode(data, JWT_SECRET, algorithm=JWT_ALGORITHM)
    # Encodes the payload (`data`) into a JWT string using the global `JWT_SECRET`
    # and `JWT_ALGORITHM`.
```

### API Endpoints (`APIRouter` Routes)

#### `GET /teams` - List Teams

Retrieves a list of all available teams.

```python
@router.get("/teams", response_model=TeamListModel)
# Decorator that registers this function as an HTTP GET endpoint at the "/teams" path.
# response_model=TeamListModel indicates that the response body will be validated against the TeamListModel Pydantic schema.
async def list_teams(adapter: AuthAdapter = Depends(get_auth_adapter)):
    # Defines an asynchronous endpoint function named `list_teams`.
    # adapter: AuthAdapter = Depends(get_auth_adapter) injects an instance of AuthAdapter.
    # `get_auth_adapter` is a dependency function that provides a database adapter.
    try:
        teams = adapter.listTeams()
        # Calls the `listTeams` method on the `AuthAdapter` to fetch team data from the database.
        return TeamListModel(teams=teams)
        # Returns the fetched teams, wrapped in the `TeamListModel` for response serialization.
    except Exception as e:
        # Catches any general exception that might occur during the process.
        raise handle_exception(e)
        # Reraises the caught exception after processing it through the custom error handler.
```

#### `POST /register` - Register User

Registers a new user in the system.

```python
@router.post("/register", response_model=dict)
# Decorator that registers this function as an HTTP POST endpoint at the "/register" path.
# response_model=dict indicates that the response will be a simple dictionary.
async def register_user(user: UserRegister, background_tasks: BackgroundTasks, adapter: AuthAdapter = Depends(get_auth_adapter)):
    # Defines an asynchronous endpoint function named `register_user`.
    # user: UserRegister: The request body is expected to conform to the UserRegister Pydantic model.
    # background_tasks: BackgroundTasks: FastAPI dependency to schedule tasks to run in the background.
    # adapter: AuthAdapter = Depends(get_auth_adapter): Injects an AuthAdapter instance.
    try:
        adapter.registerUser(username=user.UserName,
                                    email=user.Email,
                                    full_name=user.fullName,
                                    password_hash=bcrypt.hashpw(generate_secure_password().encode('utf-8'), bcrypt.gensalt()).decode('utf-8'),
                                    # Generates a secure random password, hashes it using bcrypt (with a newly generated salt),
                                    # encodes it to bytes for hashing, and then decodes it back to a string for storage.
                                    user_role=user.designation,
                                    team_id=user.TeamId)
        # Calls the `registerUser` method on the `AuthAdapter` to store the new user's information in the database.

        # Send emails as background tasks
        await registration_request_received_email(user.Email, user.UserName, background_tasks)
        # Schedules an email to the user confirming their registration request.
        await registration_request_received_email_to_admin(user.UserName, background_tasks)
        # Schedules an email to the administrator notifying them of a new registration request.

        return {"message": "Request sent successfully! Please check your mail for details."}
        # Returns a success message.
    except Exception as e:
        # Catches any general exception.
        raise handle_exception(e)
        # Reraises the exception through the custom error handler.
```

#### `POST /login` - User Login

Authenticates a user and issues JWT access and refresh tokens.

```python
@router.post("/login")
# Decorator that registers this function as an HTTP POST endpoint at the "/login" path.
async def login_user(form_data: OAuth2PasswordRequestForm = Depends(), adapter: AuthAdapter = Depends(get_auth_adapter)):
    # Defines an asynchronous endpoint function named `login_user`.
    # form_data: OAuth2PasswordRequestForm = Depends(): Extracts username and password from form data.
    # adapter: AuthAdapter = Depends(get_auth_adapter): Injects an AuthAdapter instance.
    try:
        user = adapter.login(usernameorEmail=form_data.username, password=form_data.password)
        # Calls the `login` method on the `AuthAdapter` to authenticate the user against the database.
        if not user:
            # If authentication fails (user is None or falsey),
            raise HTTPException(status_code=401, detail="Invalid username or password")
            # Raises an HTTP 401 Unauthorized error.
        access_token = create_jwt_token({"sub": user["username"], "is_admin": user.get("is_admin", False)})
        # Creates an access token with the username ("sub") and admin status in its payload.
        refresh_token = create_refresh_token({"sub": user["username"]})
        # Creates a refresh token with the username in its payload.
        terms_agreed = user.get("terms_agreed", False)
        # Retrieves the 'terms_agreed' status from the user data, defaulting to False.
        is_admin = user.get("is_admin", False)
        # Retrieves the 'is_admin' status from the user data, defaulting to False.
        team = {}
        print(user) # Debug print statement, should ideally be removed in production.
        if user.get("team_code") is not None:
            # If the user is associated with a team (has a team_code),
            team = {
                "code": user.get("team_code"),
                "name": user.get("team_name"),
                "tidyName": user.get("tidy_name"),
                "imgUrl": user.get("img_url"),
                "colorCode": user.get("primary_color_code","#af002a"),
            }
            # Populates the `team` dictionary with relevant team details.
        sports = user.get("sports", [])
        # Retrieves the 'sports' associated with the user, defaulting to an empty list.
        
        return {
            "access_token": access_token,
            "refresh_token": refresh_token,
            "token_type": "bearer", # Standard OAuth2 token type.
            "email": user.get("email", ""),
            "username": user.get("username", ""),
            "isadmin": is_admin,
            "team": team,
            "sports": sports,
            "terms_agreed": terms_agreed
        }
        # Returns the access and refresh tokens, user details, and metadata.
    except Exception as e:
        # Catches any general exception.
        raise handle_exception(e)
        # Reraises the exception through the custom error handler.
```

#### `POST /password-reset-request` - Password Reset Request

Initiates the password reset process by sending a reset link to the user's email.

```python
@router.post("/password-reset-request")
# Decorator that registers this function as an HTTP POST endpoint at the "/password-reset-request" path.
async def password_reset(usernameorEmail: str, background_tasks: BackgroundTasks, adapter: AuthAdapter = Depends(get_auth_adapter)):
    """
    This API is for password reset request. sends mail to user with reset link
    
    Args:
        usernameorEmail (str): The username or email of the user requesting a password reset.
        background_tasks (BackgroundTasks): FastAPI dependency to schedule email sending in the background.
        adapter (AuthAdapter): Injected AuthAdapter instance for database operations.
    
    Returns:
        Dict[str, Any]: A dictionary containing a confirmation message.
    """
    try:
        user = adapter.checkUserExists(usernameorEmail=usernameorEmail)
        # Checks if a user with the provided username or email exists in the database.
        if not user:
            # If no user is found,
            raise HTTPException(status_code=401, detail="Invalid username or email")
            # Raises an HTTP 401 Unauthorized error (to prevent enumeration of valid users).
        reset_token = adapter.generatePasswordResetToken(usernameorEmail=usernameorEmail)
        # Generates a unique, time-limited password reset token for the user.
        await password_reset_request_mail(to_email=user['email'], username=user['username'], reset_token=reset_token, background_tasks=background_tasks)
        # Schedules an email to be sent to the user with the generated reset token and a reset link.
        return {"message": "Password reset link sent. Please check your email."}
        # Returns a success message.
        
    except Exception as e:
        # Catches any general exception.
        raise handle_exception(e)
        # Reraises the exception through the custom error handler.
```

#### `POST /password-reset` - Password Reset

Allows a user to reset their password using a valid reset token.

```python
@router.post("/password-reset")
# Decorator that registers this function as an HTTP POST endpoint at the "/password-reset" path.
async def password_reset(token: str, password: str, adapter: AuthAdapter = Depends(get_auth_adapter)):
    """
    This API is for password reset. updates password in database
    
    Args:
        token (str): The unique reset token received by the user via email.
        password (str): The new password chosen by the user.
        adapter (AuthAdapter): Injected AuthAdapter instance for database operations.
    
    Returns:
        Dict[str, Any]: A dictionary containing a confirmation message.
    """
    try:
        adapter.updatePassword(token=token, password=password)
        # Calls the `updatePassword` method on the `AuthAdapter` to validate the token
        # and update the user's password in the database.
        return {"message": "Password reset successfully"}
        # Returns a success message.
    except Exception as e:
        # Catches any general exception (e.g., invalid or expired token).
        raise handle_exception(e)
        # Reraises the exception through the custom error handler.
```

#### `GET /terms_agreed` - Update Terms Agreement

Updates the user's status to indicate they have agreed to the terms and conditions.

```python
@router.get("/terms_agreed")
# Decorator that registers this function as an HTTP GET endpoint at the "/terms_agreed" path.
async def terms_agreed(token: str = Depends(oauth2_scheme), adapter: AuthAdapter = Depends(get_auth_adapter)):
    """
    This API is for terms agreed. updates terms agreed in database. 
    Hitting this API will update the terms agreed in database and sets terms_agreed to true.
    
    Args:
        token (str): The JWT access token provided in the Authorization header.
                     It is extracted using the `oauth2_scheme` dependency.
        adapter (AuthAdapter): Injected AuthAdapter instance for database operations.
    
    Returns:
        Dict[str, Any]: A dictionary indicating the updated terms_agreed status.
    """
    try:
        username = verify_token(token)
        # Verifies the JWT token and extracts the username (subject) from its payload.
        user = adapter.get_user_by_username_or_email(username)
        # Retrieves user details from the database using the extracted username.

        if not user:
            # If no user is found for the given username (which should ideally not happen if token is valid),
            raise HTTPException(status_code=401, detail="Invalid username or email")
            # Raises an HTTP 401 Unauthorized error.
        adapter.updateTermsAgreed(username=username)
        # Calls the `updateTermsAgreed` method on the `AuthAdapter` to mark the user as having agreed to terms.
        return {"terms_agreed": user.get("terms_agreed", True)}
        # Returns the (now true) terms_agreed status.
    except Exception as e:
        # Catches any general exception.
        raise handle_exception(e)
        # Reraises the exception through the custom error handler.
```

#### `GET /verify` - Verify Token

Verifies the authenticity and validity of a JWT token. This endpoint is typically used internally or by other services to ensure a token is valid.

```python
@router.get("/verify")
# Decorator that registers this function as an HTTP GET endpoint at the "/verify" path.
def verify_token(token: str = Depends(oauth2_scheme)):
    """
    Verify and decode JWT token.
    
    Args:
        token (str): The JWT token to verify, extracted from the Authorization header by `oauth2_scheme`.
        
    Returns:
        str: The username (subject 'sub') extracted from the token's payload if verification is successful.
        
    Raises:
        HTTPException: If the token is invalid (e.g., expired, malformed, or tampered with).
    """
    try:
        logger.debug("Verifying token: %s", token)
        # Logs the token being verified at DEBUG level.
        payload = jwt.decode(token, JWT_SECRET, algorithms=[JWT_ALGORITHM])
        # Decodes the JWT token using the global `JWT_SECRET` and `JWT_ALGORITHM`.
        # This function also automatically verifies the signature and expiration.
        logger.debug("Token decoded. Payload: %s", payload)
        # Logs the decoded payload at DEBUG level.
        return payload["sub"]
        # Returns the 'sub' (subject) claim from the payload, which is typically the username.
    except jwt.ExpiredSignatureError:
        # Catches the specific error for an expired JWT token.
        logger.warning("Token expired")
        # Logs a warning.
        raise HTTPException(status_code=401, detail="Token has expired")
        # Raises an HTTP 401 Unauthorized error with a specific detail message.
    except jwt.InvalidTokenError as e:
        # Catches any other JWT related error (e.g., invalid signature, malformed token).
        logger.error("Invalid token: %s", str(e))
        # Logs the error details.
        raise HTTPException(status_code=401, detail="Invalid token")
        # Raises an HTTP 401 Unauthorized error with a generic detail message.
```

---
