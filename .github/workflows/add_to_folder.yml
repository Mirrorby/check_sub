"""
Workflow 2 — add_to_folder.py

Reads links from Google Sheet column A (sheet "Группы"),
resolves each to a Telegram entity,
skips any channel/group the account is NOT subscribed to,
then adds all subscribed ones to the folder "База групп" (creates it if missing).

No joins are performed.
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
    DialogFilter,
    InputPeerChannel,
    InputPeerChat,
)
from telethon.tl.functions.messages import (
    GetDialogFiltersRequest,
    UpdateDialogFilterRequest,
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
LINKS_COL = "A"
HEADER_ROWS = 1

FOLDER_NAME = "База групп"

RESOLVE_DELAY = 1.0   # seconds between get_entity() calls


# ── Google Sheets helpers ──────────────────────────────────────────────────────
def open_worksheet():
    scopes = ["https://www.googleapis.com/auth/spreadsheets.readonly"]
    creds = Credentials.from_service_account_file(SERVICE_ACCOUNT_FILE, scopes=scopes)
    gc = gspread.authorize(creds)
    return gc.open_by_key(SPREADSHEET_ID).worksheet(SHEET_NAME)


def read_links(ws) -> list[str]:
    col_index = ord(LINKS_COL.upper()) - ord("A") + 1
    values = ws.col_values(col_index)
    result = []
    for i, v in enumerate(values):
        if i < HEADER_ROWS:
            continue
        v = v.strip()
        if v:
            result.append(v)
    log.info("Read %d links from sheet", len(result))
    return result


# ── Normalise link ─────────────────────────────────────────────────────────────
def normalize_link(raw: str) -> str:
    raw = raw.strip()
    if raw.startswith("@"):
        return "https://t.me/" + raw[1:]
    if raw.startswith("t.me/"):
        return "https://" + raw
    return raw


# ── Telegram: collect subscribed dialog map ────────────────────────────────────
async def build_dialog_map(client: TelegramClient) -> dict[int, object]:
    """
    Returns {entity_id: input_peer} for every dialog the account is in.
    Uses a single paginated iter_dialogs() — no per-channel requests.
    """
    dialog_map: dict[int, object] = {}
    async for dialog in client.iter_dialogs():
        entity = dialog.entity
        if isinstance(entity, (Channel, ChannelForbidden)):
            try:
                ip = await client.get_input_entity(entity)
                dialog_map[entity.id] = ip
            except Exception:
                pass
        elif isinstance(entity, (Chat, ChatForbidden)):
            try:
                ip = await client.get_input_entity(entity)
                dialog_map[entity.id] = ip
            except Exception:
                pass
    log.info("Collected %d subscribed dialogs", len(dialog_map))
    return dialog_map


# ── Telegram: resolve a single link ───────────────────────────────────────────
async def resolve_entity_id(client: TelegramClient, link: str) -> int | None:
    try:
        entity = await client.get_entity(link)
        return entity.id
    except FloodWaitError as e:
        log.warning("FloodWait %d s — sleeping", e.seconds)
        await asyncio.sleep(e.seconds + 3)
        try:
            return (await client.get_entity(link)).id
        except Exception:
            return None
    except (UsernameNotOccupiedError, UsernameInvalidError):
        log.warning("Not found: %s", link)
        return None
    except ChannelPrivateError:
        log.warning("Private: %s", link)
        return None
    except Exception as exc:
        log.warning("Error resolving %s: %s", link, exc)
        return None


# ── Telegram: folder management ───────────────────────────────────────────────
async def get_or_create_folder(client: TelegramClient, name: str) -> DialogFilter:
    result = await client(GetDialogFiltersRequest())
    filters = result.filters if hasattr(result, "filters") else result

    for f in filters:
        if isinstance(f, DialogFilter) and f.title == name:
            log.info("Found existing folder '%s' (id=%d)", name, f.id)
            return f

    # Pick unused id (2–255; id 1 is reserved for Archived)
    used_ids = {f.id for f in filters if isinstance(f, DialogFilter)}
    new_id = next(i for i in range(2, 256) if i not in used_ids)

    new_filter = DialogFilter(
        id=new_id,
        title=name,
        pinned_peers=[],
        include_peers=[],
        exclude_peers=[],
        contacts=False,
        non_contacts=False,
        groups=False,
        broadcasts=False,
        bots=False,
        exclude_muted=False,
        exclude_read=False,
        exclude_archived=False,
    )
    await client(UpdateDialogFilterRequest(id=new_id, filter=new_filter))
    log.info("Created new folder '%s' (id=%d)", name, new_id)
    return new_filter


async def save_folder_with_peers(
    client: TelegramClient,
    folder: DialogFilter,
    new_peers: list,
) -> None:
    """Merge new_peers into folder.include_peers (dedup) and save."""
    existing_ids: set[int] = set()
    for p in folder.include_peers:
        pid = (
            getattr(p, "channel_id", None)
            or getattr(p, "chat_id", None)
            or getattr(p, "user_id", None)
        )
        if pid:
            existing_ids.add(pid)

    added = 0
    for peer in new_peers:
        pid = (
            getattr(peer, "channel_id", None)
            or getattr(peer, "chat_id", None)
        )
        if pid and pid not in existing_ids:
            folder.include_peers.append(peer)
            existing_ids.add(pid)
            added += 1

    await client(UpdateDialogFilterRequest(id=folder.id, filter=folder))
    log.info("Saved folder '%s': added %d new peer(s), total %d",
             folder.title, added, len(folder.include_peers))


# ── Main ───────────────────────────────────────────────────────────────────────
async def main() -> None:
    log.info("=== add_to_folder started ===")

    # 1. Read links
    ws = open_worksheet()
    raw_links = read_links(ws)
    if not raw_links:
        log.warning("No links found. Exiting.")
        return

    async with TelegramClient(StringSession(SESSION_STRING), API_ID, API_HASH) as client:
        me = await client.get_me()
        log.info("Logged in as: %s (id=%d)", me.username or me.first_name, me.id)

        # 2. Get all current subscriptions in ONE batch call
        dialog_map = await build_dialog_map(client)

        # 3. Resolve each link → check subscription → collect input peers
        peers_to_add = []
        skipped_not_subscribed = 0
        skipped_unresolved = 0

        for raw in raw_links:
            link = normalize_link(raw)
            entity_id = await resolve_entity_id(client, link)

            if entity_id is None:
                log.info("⚠️  Cannot resolve: %s", link)
                skipped_unresolved += 1
            elif entity_id not in dialog_map:
                log.info("⏭️  Not subscribed, skip: %s", link)
                skipped_not_subscribed += 1
            else:
                peers_to_add.append(dialog_map[entity_id])
                log.info("✅ Will add to folder: %s", link)

            await asyncio.sleep(RESOLVE_DELAY)

        log.info(
            "Peers to add: %d | not subscribed: %d | unresolved: %d",
            len(peers_to_add), skipped_not_subscribed, skipped_unresolved,
        )

        if not peers_to_add:
            log.info("Nothing to add. Exiting.")
            return

        # 4. Get or create the folder
        folder = await get_or_create_folder(client, FOLDER_NAME)

        # 5. Add peers and save
        await save_folder_with_peers(client, folder, peers_to_add)

    log.info("=== Done ===")


if __name__ == "__main__":
    asyncio.run(main())
