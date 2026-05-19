import os
import time
import asyncio
from typing import Optional

import requests
from fastapi import FastAPI, Header, HTTPException
from pydantic import BaseModel
from playwright.async_api import async_playwright, Browser, Page


APPROVAL_TOKEN = os.getenv("APPROVAL_TOKEN", "change_this_approval_token_now")
CDP_URL = os.getenv("CDP_URL", "http://127.0.0.1:9222")

app = FastAPI(title="OpenClaw Human Browser Approval API")

approval_state = {
    "status": "idle",
    "message": None,
    "updated_at": time.time(),
}


class OpenRequest(BaseModel):
    url: str


class DraftPostRequest(BaseModel):
    url: str
    text: str
    require_approval: bool = True


def require_token(x_approval_token: Optional[str]):
    if x_approval_token != APPROVAL_TOKEN:
        raise HTTPException(status_code=401, detail="Invalid approval token")


def set_state(status: str, message: Optional[str] = None):
    approval_state["status"] = status
    approval_state["message"] = message
    approval_state["updated_at"] = time.time()


async def get_browser() -> Browser:
    playwright = await async_playwright().start()

    try:
        browser = await playwright.chromium.connect_over_cdp(CDP_URL)
        return browser
    except Exception as e:
        await playwright.stop()
        raise HTTPException(
            status_code=500,
            detail=f"Could not connect to Chromium CDP at {CDP_URL}: {e}",
        )


async def get_active_page(browser: Browser) -> Page:
    if not browser.contexts:
        context = await browser.new_context()
    else:
        context = browser.contexts[0]

    if context.pages:
        return context.pages[0]

    return await context.new_page()


async def wait_for_human_approval(timeout_seconds: int = 600):
    set_state("waiting_for_approval", "Waiting for human approval.")

    start = time.time()

    while time.time() - start < timeout_seconds:
        if approval_state["status"] == "approved":
            set_state("approved_consumed", "Approval received and consumed.")
            return True

        if approval_state["status"] == "denied":
            set_state("denied_consumed", "Denied by human.")
            return False

        await asyncio.sleep(1)

    set_state("timeout", "Approval timed out.")
    return False


@app.get("/health")
def health():
    return {
        "ok": True,
        "service": "openclaw-human-browser",
    }


@app.get("/status")
def status():
    return approval_state


@app.post("/approve")
def approve(x_approval_token: Optional[str] = Header(None)):
    require_token(x_approval_token)
    set_state("approved", "Human approved continuation.")
    return {"ok": True, "status": "approved"}


@app.post("/deny")
def deny(x_approval_token: Optional[str] = Header(None)):
    require_token(x_approval_token)
    set_state("denied", "Human denied continuation.")
    return {"ok": True, "status": "denied"}


@app.post("/reset")
def reset(x_approval_token: Optional[str] = Header(None)):
    require_token(x_approval_token)
    set_state("idle", "State reset.")
    return {"ok": True, "status": "idle"}


@app.post("/open")
async def open_url(
    body: OpenRequest,
    x_approval_token: Optional[str] = Header(None),
):
    require_token(x_approval_token)

    browser = await get_browser()
    page = await get_active_page(browser)

    await page.goto(body.url, wait_until="domcontentloaded")

    set_state("opened", f"Opened URL: {body.url}")

    return {
        "ok": True,
        "message": "URL opened in shared Chromium session.",
        "url": body.url,
    }


@app.post("/draft-post")
async def draft_post(
    body: DraftPostRequest,
    x_approval_token: Optional[str] = Header(None),
):
    require_token(x_approval_token)

    browser = await get_browser()
    page = await get_active_page(browser)

    await page.goto(body.url, wait_until="domcontentloaded")
    await page.wait_for_timeout(2500)

    selectors = [
        "textarea",
        "div[contenteditable='true']",
        "input[type='text']",
        "[role='textbox']",
    ]

    filled = False
    used_selector = None

    for selector in selectors:
        try:
            locator = page.locator(selector).first()
            count = await page.locator(selector).count()

            if count > 0:
                await locator.click(timeout=5000)
                await locator.fill(body.text, timeout=5000)
                filled = True
                used_selector = selector
                break
        except Exception:
            continue

    if not filled:
        set_state(
            "needs_human",
            "Could not find a safe text input. Please use noVNC manually.",
        )
        return {
            "ok": False,
            "message": "Could not find a text box. Open noVNC and handle manually.",
        }

    if body.require_approval:
        set_state(
            "waiting_for_approval",
            "Draft filled. Waiting for human approval before continuing.",
        )

        approved = await wait_for_human_approval()

        if not approved:
            return {
                "ok": False,
                "message": "Human denied or approval timed out. No final action taken.",
                "selector_used": used_selector,
            }

    return {
        "ok": True,
        "message": "Draft prepared. Final posting should still be done manually or by a separate approved action.",
        "selector_used": used_selector,
    }
