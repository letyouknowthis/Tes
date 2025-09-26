 You can customize the error response for Axum's built-in Json extractor, which produces the "Failed to deserialize..." error, using one of two primary methods:
1. Handling the Rejection in the Handler
This is the simplest approach for customizing the error response on a per-handler basis. You wrap the Json extractor in a Result in your handler signature.
The Json extractor's rejection type is axum::extract::rejection::JsonRejection. This type implements IntoResponse, but you can intercept it and convert it to your own response format.
Example Implementation
use axum::{
    extract::{Json, rejection::JsonRejection},
    http::StatusCode,
    response::{IntoResponse, Response},
};
use serde::{Deserialize, Serialize};

#[derive(Debug, Deserialize)]
struct MyPayload {
    name: String,
    age: u32,
    // The error "missing field 'options'" suggests a missing field like this
    // options: Option<String>, 
}

#[derive(Serialize)]
struct ErrorResponse {
    code: u16,
    message: String,
}

// Convert JsonRejection into your custom response
fn handle_json_rejection(rejection: JsonRejection) -> (StatusCode, Json<ErrorResponse>) {
    let (status, message) = match rejection {
        JsonRejection::JsonDataError(e) => {
            // This is the error for missing fields or type mismatch
            (StatusCode::BAD_REQUEST, format!("Invalid JSON data: {}", e))
        }
        JsonRejection::JsonSyntaxError(e) => {
            // This is the error for non-well-formed JSON
            (StatusCode::BAD_REQUEST, format!("Invalid JSON syntax: {}", e))
        }
        // Handle other possible rejections like missing content type, etc.
        _ => (
            StatusCode::INTERNAL_SERVER_ERROR,
            "Internal server error or unhandled rejection".to_string(),
        ),
    };

    let error_response = ErrorResponse {
        code: status.as_u16(),
        message,
    };

    (status, Json(error_response))
}

// Your handler function
async fn create_item(
    payload: Result<Json<MyPayload>, JsonRejection>,
) -> Response {
    match payload {
        Ok(Json(data)) => {
            // Successfully deserialized, proceed with logic
            (StatusCode::CREATED, Json(data)).into_response()
        }
        Err(rejection) => {
            // Deserialization failed, use your custom error handler
            handle_json_rejection(rejection).into_response()
        }
    }
}

// To integrate this with axum, you'd use it like:
/*
async fn main() {
    let app = Router::new().route("/items", post(create_item));
    // ... run server
}
*/

2. Creating a Custom Extractor (Recommended for Reusability)
If you want the custom error response applied everywhere you use JSON, the best practice is to create your own custom extractor, often called something like ValidatedJson or just MyJson. This involves implementing the FromRequest trait or using #[derive(FromRequest)] with the rejection option.
Steps to Create a Custom Extractor
 * Define your custom error type (e.g., ApiError). This type needs to implement IntoResponse.
 * Implement From<JsonRejection> for your custom error type. This is where you map the specific JsonRejection error variants (JsonDataError, JsonSyntaxError, etc.) into your custom structured error response.
 * Create a wrapper struct that uses #[derive(FromRequest)] to wrap axum::Json and specify your custom rejection type.
Example Implementation
1. Define Custom Error Type
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    Json,
};
use serde::Serialize;

#[derive(Serialize)]
struct ApiError {
    status_code: u16,
    error: String,
    details: Option<String>,
}

// Implement IntoResponse for your custom error
impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let status = StatusCode::from_u16(self.status_code).unwrap_or(StatusCode::BAD_REQUEST);
        (status, Json(self)).into_response()
    }
}

2. Implement From<JsonRejection>
use axum::extract::rejection::JsonRejection;

impl From<JsonRejection> for ApiError {
    fn from(rejection: JsonRejection) -> Self {
        match rejection {
            JsonRejection::JsonDataError(e) => ApiError {
                status_code: StatusCode::BAD_REQUEST.as_u16(),
                error: "InvalidPayload".to_string(),
                // Extract the specific 'missing field' error detail from the inner error
                details: Some(format!("Failed to deserialize: {}", e)), 
            },
            JsonRejection::JsonSyntaxError(e) => ApiError {
                status_code: StatusCode::BAD_REQUEST.as_u16(),
                error: "MalformedJson".to_string(),
                details: Some(format!("JSON Syntax Error: {}", e)),
            },
            // ... handle other JsonRejection variants
            _ => ApiError {
                status_code: StatusCode::INTERNAL_SERVER_ERROR.as_u16(),
                error: "ServerError".to_string(),
                details: None,
            },
        }
    }
}

3. Create the Custom Extractor
You'll need the axum-macros crate for this.
use axum_macros::FromRequest;

// Requires the `axum-macros` feature or crate
#[derive(Debug, Clone, Copy, Default, FromRequest)]
#[from_request(via(Json), rejection(ApiError))] // <--- Key line
pub struct CustomJson<T>(pub T);

4. Use it in your Handler
Now, your handler can use the custom extractor and return a Result with your custom error type:
// Your data structure
#[derive(Debug, Deserialize, Serialize)]
struct MyPayload {
    name: String,
    // Add the missing field as an Option or ensure it's in the payload
    options: Option<String>, 
}

async fn create_item_custom(
    CustomJson(payload): CustomJson<MyPayload>,
) -> Result<Json<MyPayload>, ApiError> {
    // If deserialization fails, it will automatically return your ApiError.
    // If it succeeds, you get the payload.
    println!("Received payload: {:?}", payload);
    
    // Example success response
    Ok(Json(payload))
}

/*
// To integrate this with axum, you'd use it like:
async fn main() {
    let app = Router::new().route("/items", post(create_item_custom));
    // ... run server
}
*/

This custom extractor handles the rejection internally and converts it to your ApiError, which is then converted into your desired JSON error response.
