import streamlit as st
import streamlit.components.v1 as components
import datetime
import time
import requests
from bs4 import BeautifulSoup
from zoneinfo import ZoneInfo
import json
import re

st.set_page_config(page_title="TH Taktinen Tutka", page_icon="🚕", layout="wide")

FINAVIA_API_KEY = "838062ef175f47708d566bbf5a38a710"

components.html("""
<script>
setTimeout(function(){ window.parent.location.reload(); }, 60000);
</script>
""", height=0, width=0)

if "valittu_asema" not in st.session_state:
    st.session_state.valittu_asema = "Helsinki"

st.markdown("""
<style>
#MainMenu {visibility: hidden;}
header {visibility: hidden;}
.main { background-color: #121212; }
.header-container {
    display: flex; justify-content: space-between; align-items: flex-start;
    border-bottom: 1px solid #333; padding-bottom: 15px; margin-bottom: 20px;
}
.app-title { font-size: 32px; font-weight: bold; color: #ffffff; margin-bottom: 5px; }
.time-display { font-size: 42px; font-weight: bold; color: #e0e0e0; line-height: 1; }
.taksi-card {
    background-color: #1e1e2a; color: #e0e0e0; padding: 22px;
    border-radius: 12px; margin-bottom: 20px; font-size: 20px;
    border: 1px solid #3a3a50; box-shadow: 0 4px 8px rgba(0,0,0,0.3); line-height: 1.7;
}
.card-title {
    font-size: 24px; font-weight: bold; margin-bottom: 12px;
    color: #ffffff; border-bottom: 2px solid #444; padding-bottom: 8px;
}
.taksi-link {
    color: #5bc0de; text-decoration: none; font-size: 18px;
    display: inline-block; margin-top: 12px; font-weight: bold;
}
.badge-red    { background:#7a1a1a; color:#ff9999; padding:2px 8px; border-radius:4px; font-size:16px; font-weight:bold; }
.badge-yellow { background:#5a4a00; color:#ffeb3b; padding:2px 8px; border-radius:4px; font-size:16px; font-weight:bold; }
.badge-green  { background:#1a4a1a; color:#88d888; padding:2px 8px; border-radius:4px; font-size:16px; font-weight:bold; }
.badge-blue   { background:#1a2a5a; color:#8ab4f8; padding:2px 8px; border-radius:4px; font-size:16px; font-weight:bold; }
.sold-out  { color: #ff4b4b; font-weight: bold; }
.pax-good  { color: #ffeb3b; font-weight: bold; }
.pax-ok    { color: #a3c2a3; }
.delay-bad { color: #ff9999; font-weight: bold; }
.on-time   { color: #88d888; }
.section-header {
    color: #e0e0e0; font-size: 24px; font-weight: bold;
    margin-top: 28px; margin-bottom: 10px;
    border-left: 4px solid #5bc0de; padding-left: 12px;
}
.venue-name { color: #ffffff; font-weight: bold; }
.venue-address { color: #aaaaaa; font-size: 16px; }
</style>
""", unsafe_allow_html=True)

# ── AVERIO ──────────────────────────────────────────────────────────────────

@st.cache_data(ttl=600)
def get_averio_ships():
    url = "https://averio.fi/laivat"
    headers = {
        "User-Agent": (
            "Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X) "
            "AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.0 Mobile/15E148 Safari/604.1"
        ),
        "Accept-Language": "fi-FI,fi;q=0.9",
        "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
    }
    laivat = []
    try:
        resp = requests.get(url, headers=headers, timeout=12)
        resp.raise_for_status()
        soup = BeautifulSoup(resp.text, "html.parser")
        for taulu in soup.find_all("table"):
            for rivi in taulu.find_all("tr"):
                solut = [td.get_text(strip=True) for td in rivi.find_all(["td", "th"])]
                if len(solut) < 3:
                    continue
                rivi_teksti = " ".join(solut).lower()
                if any(h in rivi_teksti for h in ["alus", "laiva", "ship", "vessel"]):
                    continue
                pax = None
                for solu in solut:
                    puhdas = re.sub(r"[^\d]", "", solu)
                    if puhdas and 50 <= int(puhdas) <= 9999:
                        pax = int(puhdas)
                        break
                nimi_kandidaatit = [s for s in solut if re.search(r"[A-Za-zÀ-ÿ]{3,}", s)]
                if not nimi_kandidaatit:
                    continue
                nimi = max(nimi_kandidaatit, key=len)
                laivat.append({
                    "ship": nimi,
                    "terminal": _tunnista_terminaali(rivi_teksti),
                    "time": _etsi_aika(solut),
                    "pax": pax,
                })
        if not laivat:
            kortit = soup.find_all("div", class_=re.compile(r"(ship|laiva|vessel|row|card|item)", re.I))
            for kortti in kortit[:6]:
                teksti = kortti.get_text(separator=" ", strip=True)
                pax = None
                for token in teksti.split():
                    puhdas = re.sub(r"[^\d]", "", token)
                    if puhdas and 50 <= int(puhdas) <= 9999:
                        pax = int(puhdas)
                        break
                laivat.append({
                    "ship": teksti[:40],
                    "terminal": _tunnista_terminaali(teksti.lower()),
                    "time": _etsi_aika(teksti.split()),
                    "pax": pax,
                })
        return laivat[:5] if laivat else [{"ship": "Averio: HTML-rakenne muuttunut", "terminal": "Tarkista manuaalisesti", "time": "-", "pax": None}]
    except Exception as e:
        return [{"ship": f"Averio-virhe: {e}", "terminal": "-", "time": "-", "pax": None}]

def _tunnista_terminaali(teksti):
    if "t2" in teksti or "lansisatama" in teksti or "länsisatama" in teksti:
        return "Länsisatama T2"
    if "t1" in teksti or "olympia" in teksti:
        return "Olympia T1"
    if "katajanokka" in teksti:
        return "Katajanokka"
    if "vuosaari" in teksti:
        return "Vuosaari (rahti)"
    return "Tarkista"

def _etsi_aika(osat):
    for osa in osat:
        m = re.search(r"\b([0-2]?\d:[0-5]\d)\b", str(osa))
        if m:
            return m.group(1)
    return "-"

def _pax_arvio(pax):
    if pax is None:
        return "Ei tietoa", "pax-ok"
    autoa = round(pax * 0.025)
    if pax >= 400:
        return f"🔥 {pax} matkustajaa (~{autoa} autoa, ERINOMAINEN)", "pax-good"
    if pax >= 200:
        return f"✅ {pax} matkustajaa (~{autoa} autoa)", "pax-ok"
    return f"⬇️ {pax} matkustajaa (~{autoa} autoa, matala)", "pax-ok"

# ── HELSINGIN SATAMA ─────────────────────────────────────────────────────────

@st.cache_data(ttl=600)
def get_port_schedule():
    url = "https://www.portofhelsinki.fi/matkustajille/matkustajatietoa/lahtevat-ja-saapuvat-matkustajalaivat/#tabs-2"
    headers = {"User-Agent": "Mozilla/5.0"}
    try:
        resp = requests.get(url, headers=headers, timeout=12)
        resp.raise_for_status()
        soup = BeautifulSoup(resp.text, "html.parser")
        lista = []
        for rivi in soup.find_all("tr"):
            solut = rivi.find_all("td")
            if len(solut) >= 4:
                aika = solut[0].get_text(strip=True)
                laiva = solut[1].get_text(strip=True)
                terminaali = solut[3].get_text(strip=True) if len(solut) > 3 else "?"
                if aika and laiva and re.match(r"\d{1,2}:\d{2}", aika):
                    lista.append({"time": aika, "ship": laiva, "terminal": terminaali})
        return lista[:6] if lista else []
    except Exception as e:
        return [{"time": "-", "ship": f"Virhe: {e}", "terminal": "-"}]

# ── JUNAT ────────────────────────────────────────────────────────────────────

@st.cache_data(ttl=50)
def get_trains(asema_nimi):
    nykyhetki = datetime.datetime.now(ZoneInfo("Europe/Helsinki"))
    koodit = {"Helsinki": "HKI", "Pasila": "PSL", "Tikkurila": "TKL"}
    koodi = koodit.get(asema_nimi, "HKI")
    url = (
        f"https://rata.digitraffic.fi/api/v1/live-trains/station/{koodi}"
        "?arrived_trains=0&arriving_trains=25&train_categories=Long-distance"
    )
    kaupungit = {
        "ROV": "Rovaniemi", "OUL": "Oulu", "TPE": "Tampere", "TKU": "Turku",
        "KJA": "Kajaani", "JNS": "Joensuu", "YV": "Ylivieska", "VAA": "Vaasa",
        "JY": "Jyvaskyla", "KUO": "Kuopio", "POR": "Pori", "SJK": "Seinajoki",
        "LVT": "Lappeenranta", "KOK": "Kokkola", "KEM": "Kemi", "KTI": "Kittila",
    }
    tulos = []
    try:
        resp = requests.get(url, timeout=10)
        resp.raise_for_status()
        data = resp.json()
        for juna in data:
            if juna.get("cancelled"):
                continue
            tyyppi = juna.get("trainType", "")
            numero = juna.get("trainNumber", "")
            nimi = f"{tyyppi} {numero}"
            lahto = juna["timeTableRows"][0]["stationShortCode"]
            if lahto in ("HKI", "PSL", "TKL"):
                continue
            lahto_kaupunki = kaupungit.get(lahto, lahto)
            aika_obj = aika_str = None
            viive = 0
            for rivi in juna["timeTableRows"]:
                if rivi["stationShortCode"] == koodi and rivi["type"] == "ARRIVAL":
                    raaka = rivi.get("liveEstimateTime") or rivi.get("scheduledTime", "")
                    try:
                        aika_obj = datetime.datetime.strptime(raaka[:16], "%Y-%m-%dT%H:%M")
                        aika_obj = aika_obj.replace(tzinfo=datetime.timezone.utc).astimezone(ZoneInfo("Europe/Helsinki"))
                        if aika_obj < nykyhetki - datetime.timedelta(minutes=3):
                            aika_obj = None
                        else:
                            aika_str = aika_obj.strftime("%H:%M")
                    except Exception:
                        pass
                    viive = rivi.get("differenceInMinutes", 0)
                    break
            if aika_str and aika_obj:
                tulos.append({"train": nimi, "origin": lahto_kaupunki, "time": aika_str, "dt": aika_obj, "delay": viive if viive > 0 else 0})
        tulos.sort(key=lambda k: k["dt"])
        return tulos[:6]
    except Exception as e:
        return [{"train": "API-virhe", "origin": str(e)[:30], "time": "-", "dt": None, "delay": 0}]

# ── LENNOT (Uusi Finavia + OpenSky varajärjestelmä) ──────────────────────────

# ICAO-koodi → kaupunki (OpenSky palauttaa ICAO-koodit)
ICAO_KAUPUNGIT = {
    "EDDF": "Frankfurt", "EGLL": "London Heathrow", "LFPG": "Pariisi CDG",
    "LEMD": "Madrid", "LIRF": "Rooma", "EHAM": "Amsterdam", "EPWA": "Varsova",
    "ESSA": "Tukholma ARN", "ENGM": "Oslo", "EKCH": "Kööpenhamina",
    "LOWW": "Wien", "LSZH": "Zürich", "LSGG": "Geneve", "EGKK": "London Gatwick",
    "UUEE": "Moskova SVO", "LTFM": "Istanbul", "OMDB": "Dubai",
    "VHHH": "Hongkong", "RJTT": "Tokio", "KJFK": "New York JFK",
    "EVRA": "Riika", "EYVI": "Vilna", "EETN": "Tallinna",
    "UMMS": "Minsk", "UKBB": "Kiova", "OEJN": "Jeddah",
}

def _finavia_parse(data):
    if isinstance(data, list):
        return data
    if isinstance(data, dict):
        for avain in ("arr", "flights", "body"):
            k = data.get(avain)
            if isinstance(k, list):
                return k
            if isinstance(k, dict):
                for ala in ("arr", "flight"):
                    if isinstance(k.get(ala), list):
                        return k[ala]
    return []

def _build_flight_list(saapuvat, laajarunko, source=""):
    tulos = []
    for lento in saapuvat:
        nro    = lento.get("fltnr") or lento.get("flightNumber", "??")
        kohde  = lento.get("route_n_1") or lento.get("airport", "Tuntematon")
        aika_r = str(lento.get("sdt") or lento.get("scheduledTime", ""))
        actype = str(lento.get("actype") or lento.get("aircraftType", ""))
        status = str(lento.get("prt_f") or lento.get("statusInfo", "Odottaa"))
        aika   = aika_r[11:16] if "T" in aika_r else aika
