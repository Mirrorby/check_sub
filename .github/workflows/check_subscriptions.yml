"""
Workflow 1 — check_subscriptions.py

Reads links from Google Sheet column A (starting row 2, sheet "Группы"),
checks if the Telegram account is subscribed to each channel/group,
writes ✅ or ❌ to column B.

No joins are performed — read-only on the Telegram side.
"""

import asyncio
import logging
import os
import sys

import gspread
from google.oauth2.service_account import Credentials
from telethon import TelegramClient
from telethon.sessions import StringSession
from telethon.tl.types import (
    Channel, Chat,
    ChannelForbidden, ChatForbidden,
)
from telethon.errors import (
    FloodWaitError,
    UsernameNotOccupiedError,
    ChannelPrivateError,
    UsernameInvalidError,
)

# ── Logging ────────────────────────────────────────────────────────────────────
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    handlers=[logging.StreamHandler(sys.stdout)],
)
log = logging.getLogger(__name__)

# ── Config ─────────────────────────────────────────────────────────────────────
API_ID = int(os.environ["TELEGRAM_API_ID"])
API_HASH = os.environ["TELEGRAM_API_HASH"]
SESSION_STRING = os.environ["TELEGRAM_SESSION_STRING"]
SPREADSHEET_ID = os.environ["GOOGLE_SPREADSHEET_ID"]
SERVICE_ACCOUNT_FILE = os.environ.get("GOOGLE_SERVICE_ACCOUNT_FILE", "service_account.json")

SHEET_NAME = "Группы"
LINKS_COL = "A"       # links
STATUS_COL = "B"      # where to write ✅ / ❌
HEADER_ROWS = 1       # skip first row

STATUS_SUBSCRIBED = "✅ подписан"
STATUS_NOT_SUBSCRIBED = "❌ не подписан"
STATUS_UNKNOWN = "⚠️ недоступен"

# Delay between entity resolutions (seconds) — keeps us safe from FloodWait
RESOLVE_DELAY = 1.0


# ── Google Sheets helpers ──────────────────────────────────────────────────────
def open_worksheet():
    scopes = [
        "https://www.googleapis.com/auth/spreadsheets",
    ]
    creds = Credentials.from_service_account_file(SERVICE_ACCOUNT_FILE, scopes=scopes)
    gc = gspread.authorize(creds)
    spreadsheet = gc.open_by_key(SPREADSHEET_ID)
    return spreadsheet.worksheet(SHEET_NAME)


def read_links(ws) -> list[tuple[int, str]]:
    """Return list of (row_number, url) starting after header rows."""
    col_index = ord(LINKS_COL.upper()) - ord("A") + 1
    values = ws.col_values(col_index)
    result = []
    for i, v in enumerate(values):
        row = i + 1
        if row <= HEADER_ROWS:
            continue
        v = v.strip()
        if v:
            result.append((row, v))
    log.info("Read %d links from sheet", len(result))
    return result


def write_statuses(ws, updates: list[tuple[int, str]]) -> None:
    """Batch-write status values to column B. updates = [(row, value), ...]"""
    col_index = ord(STATUS_COL.upper()) - ord("A") + 1
    col_letter = STATUS_COL.upper()

    # Build a batch update
    cell_list = []
    for row, value in updates:
        cell = gspread.Cell(row, col_index, value)
        cell_list.append(cell)

    ws.update_cells(cell_list, value_input_option="USER_ENTERED")
    log.info("Wrote %d status cells to sheet", len(cell_list))


# ── Telegram helpers ───────────────────────────────────────────────────────────
def normalize_link(raw: str) -> str:
    """Ensure link is in https://t.me/... form."""
    raw = raw.strip()
    if raw.startswith("@"):
        return "https://t.me/" + raw[1:]
    if raw.startswith("t.me/"):
        return "https://" + raw
    return raw


async def build_subscribed_ids(client: TelegramClient) -> set[int]:
    """
    Collect all entity IDs the account is currently subscribed to.
    Uses get_dialogs() — one paginated call, no per-channel requests.
    """
    subscribed: set[int] = set()
    async for dialog in client.iter_dialogs():
        entity = dialog.entity
        if isinstance(entity, (Channel, Chat, ChannelForbidden, ChatForbidden)):
            subscribed.add(entity.id)
    log.info("Account is subscribed to %d channels/groups/chats", len(subscribed))
    return subscribed


async def resolve_entity_id(client: TelegramClient, link: str) -> int | None:
    """Resolve a t.me link to a Telegram entity ID. Returns None on failure."""
    try:
        entity = await client.get_entity(link)
        return entity.id
    except FloodWaitError as e:
        log.warning("FloodWait %d s on resolving %s — sleeping", e.seconds, link)
        await asyncio.sleep(e.seconds + 3)
        # retry once
        try:
            entity = await client.get_entity(link)
            return entity.id
        except Exception:
            return None
    except (UsernameNotOccupiedError, UsernameInvalidError):
        log.warning("Username not found: %s", link)
        return None
    except ChannelPrivateError:
        log.warning("Private / inaccessible: %s", link)
        return None
    except Exception as exc:
        log.warning("Cannot resolve %s: %s", link, exc)
        return None


# ── Main ───────────────────────────────────────────────────────────────────────
async def main() -> None:
    log.info("=== check_subscriptions started ===")

    # 1. Read links from sheet
    ws = open_worksheet()
    rows = read_links(ws)
    if not rows:
        log.warning("No links found. Exiting.")
        return

    # 2. Connect to Telegram
    async with TelegramClient(StringSession(SESSION_STRING), API_ID, API_HASH) as client:
        me = await client.get_me()
        log.info("Logged in as: %s (id=%d)", me.username or me.first_name, me.id)

        # 3. Get all current subscriptions in ONE batch call
        subscribed_ids = await build_subscribed_ids(client)

        # 4. For each link — resolve ID and compare
        updates: list[tuple[int, str]] = []
        for row, raw_link in rows:
            link = normalize_link(raw_link)
            entity_id = await resolve_entity_id(client, link)

            if entity_id is None:
                status = STATUS_UNKNOWN
            elif entity_id in subscribed_ids:
                status = STATUS_SUBSCRIBED
            else:
                status = STATUS_NOT_SUBSCRIBED

            log.info("Row %d | %-55s → %s", row, link, status)
            updates.append((row, status))

            await asyncio.sleep(RESOLVE_DELAY)

    # 5. Write all results back to the sheet in one batch
    write_statuses(ws, updates)

    subscribed_count = sum(1 for _, s in updates if s == STATUS_SUBSCRIBED)
    not_subscribed_count = sum(1 for _, s in updates if s == STATUS_NOT_SUBSCRIBED)
    unknown_count = sum(1 for _, s in updates if s == STATUS_UNKNOWN)

    log.info("=== Done ===")
    log.info("✅ subscribed:     %d", subscribed_count)
    log.info("❌ not subscribed: %d", not_subscribed_count)
    log.info("⚠️  unavailable:   %d", unknown_count)


if __name__ == "__main__":
    asyncio.run(main())
