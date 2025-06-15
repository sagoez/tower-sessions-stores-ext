<h1 align="center">
    tower-sessions-ext-moka-store
</h1>

<p align="center">
    Moka session store for `tower-sessions-ext`.
</p>

> [!NOTE]
> This is a maintained fork of the original implementation. The repository has been migrated from `maxcountryman` to `sagoez` to ensure continued maintenance and support.

## 🤸 Usage

```rust
use std::net::SocketAddr;

use axum::{response::IntoResponse, routing::get, Router};
use serde::{Deserialize, Serialize};
use time::Duration;
use tower_sessions_ext::{CachingSessionStore, Expiry, Session, SessionManagerLayer};
use tower_sessions_ext_moka_store::MokaStore;
use tower_sessions_ext_sqlx_store::{sqlx::SqlitePool, SqliteStore};

const COUNTER_KEY: &str = "counter";

#[derive(Serialize, Deserialize, Default)]
struct Counter(usize);

async fn handler(session: Session) -> impl IntoResponse {
    let counter: Counter = session.get(COUNTER_KEY).await.unwrap().unwrap_or_default();
    session.insert(COUNTER_KEY, counter.0 + 1).await.unwrap();
    format!("Current count: {}", counter.0)
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let pool = SqlitePool::connect(":memory:").await?;

    let sqlite_store = SqliteStore::new(pool);
    sqlite_store.migrate().await?;

    let moka_store = MokaStore::new(Some(2000));
    let caching_store = CachingSessionStore::new(moka_store, sqlite_store);

    let session_layer = SessionManagerLayer::new(caching_store)
        .with_secure(false)
        .with_expiry(Expiry::OnInactivity(Duration::seconds(10)));

    let app = Router::new().route("/", get(handler)).layer(session_layer);

    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    let listener = tokio::net::TcpListener::bind(&addr).await?;
    axum::serve(listener, app.into_make_service()).await?;

    Ok(())
}

## 🙏 Acknowledgments

Special thanks to [maxcountryman](https://github.com/maxcountryman) for the original implementation and groundwork for this project.
