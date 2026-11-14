import os
import smtplib
from datetime import datetime, timedelta, timezone
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
import feedparser
from google import genai
from google.genai import types

# Target email address
EMAIL_ADDRESS = "maxoromurillo@gmail.com"

# Direct XML/RSS Feeds (Tier 1, 3, and 5)
FEEDS = {
    "EC Press Corner (Competition)": "https://ec.europa.eu/commission/presscorner/api/rss?language=en&policy=COMP",
    "CURIA (CJEU Press Releases)": "https://curia.europa.eu/jcms/upload/docs/application/rss+xml/curia_press_en.xml",
    "Kluwer Competition Law Blog": "https://competitionlawblog.kluwercompetitionlaw.com/feed/",
    "Chillin'Competition": "https://chillingcompetition.com/feed/",
    "The Platform Law Blog (Geradin)": "https://theplatformlaw.blog/feed/",
    "D'Kart Antitrust Blog": "https://www.d-kart.de/en/feed/",
    "European Law Blog": "https://europeanlawblog.eu/feed/",
    "EU Law Live": "https://eulawlive.com/feed/",
    "Concurrences": "https://www.concurrences.com/en/spip.php?page=backend",
    "Bruegel": "https://www.bruegel.org/rss.xml",
    "Politico EU Competition": "https://www.politico.eu/section/competition/feed/",
    "Expansion Juridico": "https://e00-expansion.uecdn.es/rss/juridico.xml",
}

def fetch_rss_digest():
    now_utc = datetime.now(timezone.utc)
    # Monday (weekday == 0) looks back 72 hours (Friday morning through Sunday night)
    lookback_hours = 72 if now_utc.weekday() == 0 else 24
    cutoff = now_utc - timedelta(hours=lookback_hours)
    collected = []

    for source_name, url in FEEDS.items():
        try:
            feed = feedparser.parse(url)
            for entry in feed.entries[:15]:
                pub_parsed = getattr(entry, "published_parsed", None) or getattr(entry, "updated_parsed", None)
                if pub_parsed:
                    entry_dt = datetime(*pub_parsed[:6], tzinfo=timezone.utc)
                    if entry_dt < cutoff:
                        continue
                collected.append(
                    f"SOURCE: {source_name}\n"
                    f"TITLE: {entry.get('title', '').strip()}\n"
                    f"URL: {entry.get('link', '').strip()}\n"
                    f"SNIPPET: {entry.get('summary', '')[:600].strip()}"
                )
        except Exception as e:
            collected.append(f"SOURCE: {source_name} (Feed read warning: {e})")

    return lookback_hours, "\n---\n".join(collected)

def build_briefing_with_hydrogen(lookback_hours, rss_digest):
    client = genai.Client(api_key=os.environ["GEMINI_API_KEY"])
    today_date = datetime.now(timezone.utc).strftime("%A, %d %B %Y")
    is_monday = datetime.now(timezone.utc).weekday() == 0

    window_instruction = (
        "Today is MONDAY: analyze the last 72 hours (Friday afternoon through Sunday) AND include a brief preview of the CJEU judicial calendar for the week ahead."
        if is_monday
        else f"Analyze strictly developments from the past {lookback_hours} hours."
    )

    prompt = f"""
    Today is {today_date}. {window_instruction}
    You are a Senior Competition Law Knowledge Lawyer preparing a high-signal 3-minute morning briefing.

    Use the ingested RSS feed items below AND use Google Search to check the latest press releases from the past {lookback_hours} hours for:
    - EU Institutions: DG COMP (ec.europa.eu/competition), CJEU Curia, EUR-Lex Official Journal competition notices.
    - National Competition Authorities & Courts: Spain (CNMC cnmc.es & Audiencia Nacional/Tribunal Supremo competition rulings), Germany (Bundeskartellamt), France (Autorité de la concurrence), Italy (AGCM), Netherlands (ACM), and UK (CMA).
    - Key Voices & Firm Alerts: Damien Geradin, Assimakis Komninos, Alfonso Lamadrid, Pablo Ibáñez Colomo, Cristina Caffarra, Fiona Scott Morton, Daniel Sarmiento, Uría Menéndez, Cuatrecasas, Garrigues, Pérez-Llorca, Cleary Gottlieb, Freshfields, White & Case.

    STRICT RULES:
    1. Output ONLY clean HTML suitable for an email body (use <h2>, <h3>, <ul>, <li>, <b>, <p>, and <a href="..."> tags). Do not wrap in markdown code fences.
    2. GROUNDING: Never invent legal holdings. Cite exact case numbers where available and attach a clickable hyperlink to every single item.

    SECTIONS:
    I. THE 60-SECOND PARTNER BRIEF (Top 1-2 most significant developments and why they matter in practice)
    II. EU COMPETITION, MERGER CONTROL, STATE AID, DMA & FSR (DG COMP, CJEU & Official Journal)
    III. NATIONAL COMPETITION AUTHORITIES & COURTS (CNMC/Spain, BKartA/Germany, AdlC/France, AGCM/Italy, ACM/Netherlands, CMA/UK)
    IV. DOCTRINE, BLOGS & PRACTITIONER DEBATES (Kluwer, EU Law Live, Concurrences, Chillin'Competition, Platform Law, D'Kart, European Law Blog)
    V. EU MACRO, FINANCIAL & M&A CONTEXT (Bruegel, Politico Europe, FT, Expansión Jurídico)

    INGESTED RSS ITEMS (PAST {lookback_hours} HOURS):
    {rss_digest}
    """

    # Explicitly locked to Gemini 4 Hydrogen with Google Search grounding enabled
    response = client.models.generate_content(
        model="gemini-4-hydrogen",
        contents=prompt,
        config=types.GenerateContentConfig(
            temperature=0.1,
            tools=[types.Tool(google_search=types.GoogleSearch())],
        ),
    )
    return response.text

def send_gmail(html_content):
    password = os.environ["GMAIL_APP_PASSWORD"]
    today_str = datetime.now().strftime("%a %d %b %Y")

    msg = MIMEMultipart("alternative")
    msg["Subject"] = f"[Competition Law & EU Briefing] — {today_str}"
    msg["From"] = EMAIL_ADDRESS
    msg["To"] = EMAIL_ADDRESS

    msg.attach(MIMEText(html_content, "html", "utf-8"))

    with smtplib.SMTP_SSL("smtp.gmail.com", 465) as server:
        server.login(EMAIL_ADDRESS, password)
        server.sendmail(EMAIL_ADDRESS, EMAIL_ADDRESS, msg.as_string())

if __name__ == "__main__":
    hours, digest = fetch_rss_digest()
    html_report = build_briefing_with_hydrogen(hours, digest)
    send_gmail(html_report)
