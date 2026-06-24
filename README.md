curl -sL -o collect.py https://raw.githubusercontent.com/sc9rrtk2c8-star/collect.py/main/README.md
python collect.py
# -*- coding: utf-8 -*-
import sys, os
import re
import json
import time
import requests
from bs4 import BeautifulSoup

JCD = {
    "桐生": 1, "戸田": 2, "江戸川": 3, "平和島": 4, "多摩川": 5, "浜名湖": 6,
    "蒲郡": 7, "常滑": 8, "津": 9, "三国": 10, "びわこ": 11, "住之江": 12,
    "尼崎": 13, "鳴門": 14, "丸亀": 15, "児島": 16, "宮島": 17, "徳山": 18,
    "下関": 19, "若松": 20, "芦屋": 21, "福岡": 22, "唐津": 23, "大村": 24,
}
JCD_NAME = {v: k for k, v in JCD.items()}

HEADERS = {
    "User-Agent": ("Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
                   "AppleWebKit/537.36 (KHTML, like Gecko) "
                   "Chrome/124.0 Safari/537.36 personal-research"),
    "Accept-Language": "ja,en;q=0.8",
}
URL_RACELIST = "https://www.boatrace.jp/owpc/pc/race/racelist"
URL_BEFORE = "https://www.boatrace.jp/owpc/pc/race/beforeinfo"
URL_ODDS3T = "https://www.boatrace.jp/owpc/pc/race/odds3t"


def _num(s):
    if not s:
        return None
    m = re.search(r"-?\d+(?:\.\d+)?", s)
    return float(m.group()) if m else None


def _floats(s):
    return [float(x) for x in re.findall(r"-?\d+(?:\.\d+)?", s or "")]


def _get(url, jcd, rno, hd, session):
    params = {"rno": int(rno), "jcd": f"{int(jcd):02d}", "hd": str(hd)}
    r = session.get(url, params=params, headers=HEADERS, timeout=15)
    r.raise_for_status()
    r.encoding = "utf-8"
    return r.text


def parse_racelist(html, jcd, rno, hd):
    soup = BeautifulSoup(html, "html.parser")
    racers = []
    seen = set()
    for img in soup.find_all("img", src=re.compile(r"racerphoto")):
        mt = re.search(r"racerphoto/(\d+)\.", img.get("src", ""))
        toban = mt.group(1) if mt else None
        if not toban or toban in seen:
            continue
        seen.add(toban)
        tr = img.find_parent("tr")
        if tr is None:
            continue
        tds = tr.find_all("td")
        info_td = None
        for td in tds:
            txt = td.get_text(" ", strip=True)
            if toban in txt and re.search(r"\b(A1|A2|B1|B2)\b", txt):
                info_td = td
                break
        if info_td is None:
            continue
        info = info_td.get_text(" ", strip=True)
        name = None
        for a in info_td.find_all("a", href=re.compile(r"toban=")):
            t = a.get_text(strip=True)
            if t:
                name = t
                break
        m_cls = re.search(r"(A1|A2|B1|B2)", info)
        m_age = re.search(r"(\d+)歳", info)
        m_wt = re.search(r"([\d.]+)kg", info)
        m_br = re.search(r"([^\s/]+)/([^\s/]+)\s+\d+歳", info)
        sib = info_td.find_next_siblings("td")

        def sx(i):
            return sib[i].get_text(" ", strip=True) if i < len(sib) else ""

        flst = sx(0)
        m_f = re.search(r"F(\d+)", flst)
        m_l = re.search(r"L(\d+)", flst)
        m_st = re.search(r"(\d\.\d{2})", flst)
        nat = _floats(sx(1))
        loc = _floats(sx(2))
        mot = _floats(sx(3))
        boa = _floats(sx(4))

        def g(lst, i):
            return lst[i] if i < len(lst) else None

        racers.append({
            "toban": toban, "name": name,
            "class": m_cls.group(1) if m_cls else None,
            "branch": m_br.group(1) if m_br else None,
            "origin": m_br.group(2) if m_br else None,
            "age": int(m_age.group(1)) if m_age else None,
            "weight": float(m_wt.group(1)) if m_wt else None,
            "f_count": int(m_f.group(1)) if m_f else None,
            "l_count": int(m_l.group(1)) if m_l else None,
            "st_avg": float(m_st.group(1)) if m_st else None,
            "nat_win": g(nat, 0), "nat_2": g(nat, 1), "nat_3": g(nat, 2),
            "loc_win": g(loc, 0), "loc_2": g(loc, 1), "loc_3": g(loc, 2),
            "motor_no": int(g(mot, 0)) if g(mot, 0) is not None else None,
            "motor_2": g(mot, 1), "motor_3": g(mot, 2),
            "boat_no": int(g(boa, 0)) if g(boa, 0) is not None else None,
            "boat_2": g(boa, 1), "boat_3": g(boa, 2),
        })
    for i, r in enumerate(racers, start=1):
        r["lane"] = i
    return racers


def parse_before(html):
    soup = BeautifulSoup(html, "html.parser")
    racer_table = None
    for t in soup.find_all("table"):
        if t.find("img", src=re.compile(r"racerphoto")):
            racer_table = t
            break
    racers = []
    if racer_table:
        for tb in racer_table.find_all("tbody"):
            img = tb.find("img", src=re.compile(r"racerphoto"))
            if not img:
                continue
            mt = re.search(r"racerphoto/(\d+)\.", img.get("src", ""))
            toban = mt.group(1) if mt else None
            trs = tb.find_all("tr")
            first = trs[0] if trs else None
            cells = [td.get_text(" ", strip=True) for td in (first.find_all("td") if first else [])]

            def cell(i):
                return cells[i] if i < len(cells) else ""

            weight_adj = None
            for tr in trs[1:]:
                v = _num(tr.get_text(" ", strip=True))
                if v is not None and v <= 5:
                    weight_adj = v
                    break
            racers.append({
                "toban": toban,
                "ex_time": _num(cell(4)),
                "tilt": _num(cell(5)),
                "propeller_new": "新" in (cell(6) or ""),
                "parts": cell(7).strip(),
                "weight_adj": weight_adj,
            })
    start = []
    for t in soup.find_all("table"):
        if "スタート展示" not in t.get_text():
            continue
        for tr in t.find_all("tr"):
            bimg = tr.find("img", src=re.compile(r"img_boat2_(\d)"))
            if not bimg:
                continue
            lane = int(re.search(r"img_boat2_(\d)", bimg["src"]).group(1))
            txt = tr.get_text(" ", strip=True)
            mst = re.search(r"\.(\d{2})", txt)
            start.append({
                "course": len(start) + 1,
                "lane": lane,
                "st": float("0." + mst.group(1)) if mst else None,
                "is_F": "F" in txt,
                "is_L": bool(re.search(r"(?<!\w)L(?!\w)", txt)),
            })
        break
    full = soup.get_text(" ", strip=True)

    def gw(pat):
        m = re.search(pat, full)
        return float(m.group(1)) if m else None

    weather = {
        "air_temp": gw(r"気温\s*([\d.]+)"),
        "water_temp": gw(r"水温\s*([\d.]+)"),
        "wind_speed": gw(r"風速\s*([\d.]+)"),
        "wave_height": gw(r"波高\s*([\d.]+)"),
        "condition": next((c for c in ("晴", "曇", "雨", "雪") if c in full), None),
    }
    return racers, start, weather


def parse_odds3t(html):
    soup = BeautifulSoup(html, "html.parser")
    best_vals = []
    for t in soup.find_all("table"):
        vals = []
        for td in t.find_all("td"):
            tx = td.get_text(strip=True).replace(",", "")
            if re.fullmatch(r"\d+\.\d+", tx):
                vals.append(float(tx))
            elif re.fullmatch(r"\d+", tx) and int(tx) > 6:
                vals.append(float(tx))
        if len(vals) > len(best_vals):
            best_vals = vals
    odds = {}
    if len(best_vals) >= 120:
        for i, v in enumerate(best_vals[:120]):
            p = i // 6
            first = i % 6 + 1
            others = [x for x in range(1, 7) if x != first]
            second = others[p // 4]
            rem = [x for x in others if x != second]
            third = rem[p % 4]
            odds[f"{first}-{second}-{third}"] = v
    return odds


def build_race(jcd, rno, hd):
    sess = requests.Session()
    card = parse_racelist(_get(URL_RACELIST, jcd, rno, hd, sess), jcd, rno, hd)
    time.sleep(1.0)
    try:
        bf_racers, start, weather = parse_before(_get(URL_BEFORE, jcd, rno, hd, sess))
    except Exception as e:
        print(f"[warn] beforeinfo 取得失敗(締切前か?): {e}")
        bf_racers, start, weather = [], [], {}
    try:
        odds = parse_odds3t(_get(URL_ODDS3T, jcd, rno, hd, sess))
    except Exception as e:
        print(f"[warn] odds 取得失敗: {e}")
        odds = {}
    bf_by_toban = {r["toban"]: r for r in bf_racers}
    start_by_lane = {s["lane"]: s for s in start}
    racers = []
    for r in card:
        bf = bf_by_toban.get(r["toban"], {})
        st = start_by_lane.get(r["lane"], {})
        racers.append({
            "lane": r["lane"], "toban": r["toban"], "name": r.get("name"),
            "class": r.get("class"), "branch": r.get("branch"),
            "origin": r.get("origin"), "age": r.get("age"), "weight": r.get("weight"),
            "nat_win": r.get("nat_win"), "nat_2": r.get("nat_2"), "nat_3": r.get("nat_3"),
            "loc_win": r.get("loc_win"), "loc_2": r.get("loc_2"), "loc_3": r.get("loc_3"),
            "motor_no": r.get("motor_no"), "motor_2": r.get("motor_2"), "motor_3": r.get("motor_3"),
            "boat_no": r.get("boat_no"), "boat_2": r.get("boat_2"), "boat_3": r.get("boat_3"),
            "f_count": r.get("f_count"), "l_count": r.get("l_count"), "st_avg": r.get("st_avg"),
            "ex_time": bf.get("ex_time"), "tilt": bf.get("tilt"),
            "parts": bf.get("parts"), "propeller_new": bf.get("propeller_new"),
            "weight_adj": bf.get("weight_adj"),
            "entry_course": st.get("course"), "ex_st": st.get("st"),
            "ex_is_F": st.get("is_F"), "ex_is_L": st.get("is_L"),
        })
    return {
        "jcd": jcd, "place": JCD_NAME.get(int(jcd)), "rno": rno, "hd": hd,
        "weather": weather, "racers": racers, "odds": odds,
    }


PARTS_ALL = ["ピストン", "リング", "電気", "キャブ", "シリンダ", "シャフト", "ギヤ", "キャリボ"]
PARTS_HEAVY = ["ピストン", "シリンダ", "シャフト", "キャブ"]
CLASS_RANK = {"A1": 4, "A2": 3, "B1": 2, "B2": 1}

def add_dynamic(race):
    rs = race.get("racers", [])
    w = race.get("weather", {}) or {}
    wind = w.get("wind_speed") or 0.0
    wave = w.get("wave_height") or 0.0
    exs = [(r["lane"], r.get("ex_time")) for r in rs if r.get("ex_time") is not None]
    order = sorted(exs, key=lambda x: x[1])
    ex_rank = {ln: i + 1 for i, (ln, _) in enumerate(order)}
    for r in rs:
        parts = r.get("parts") or ""
        cls = CLASS_RANK.get(r.get("class"), 2)
        rank = ex_rank.get(r["lane"], 6)
        r["dyn"] = {
            "tilt": r.get("tilt"),
            "parts_any": int(any(k in parts for k in PARTS_ALL)),
            "parts_heavy": int(any(k in parts for k in PARTS_HEAVY)),
            "propeller_new": int(bool(r.get("propeller_new"))),
            "wind": wind,
            "wave": wave,
            "ex_rank": rank,
            "motor_surprise": round((3 - cls) * (3.5 - rank) / 6.0, 3),
        }
    return race


DATASET = "dataset.json"

def load_dataset():
    if os.path.exists(DATASET):
        try:
            return json.load(open(DATASET, encoding="utf-8"))
        except Exception:
            return []
    return []

def save_dataset(rows):
    json.dump(rows, open(DATASET, "w", encoding="utf-8"), ensure_ascii=False, indent=1)

def race_id(jcd, hd, rno):
    return f"{hd}-{jcd:02d}-{rno:02d}"

def collect(jcd, hd, only_rno=None):
    rows = load_dataset()
    have = {r["race_id"] for r in rows}
    targets = [only_rno] if only_rno else list(range(1, 13))
    added = 0
    for rno in targets:
        rid = race_id(jcd, hd, rno)
        if rid in have:
            print(f"skip {rid}")
            continue
        try:
            race = add_dynamic(build_race(jcd, rno, hd))
            rows.append({
                "race_id": rid, "jcd": jcd, "hd": hd, "rno": rno,
                "place": race.get("place"), "racers": race.get("racers"),
                "weather": race.get("weather"), "odds": race.get("odds"),
                "result": None,
            })
            added += 1
            print(f"OK   {rid}  {len(race.get('racers',[]))}艇 odds={len(race.get('odds',{}))}点")
        except Exception as e:
            print(f"FAIL {rid}: {e}")
        time.sleep(2.5)
    save_dataset(rows)
    print(f"--- {added}件追加 / 合計 {len(rows)}件 ---")

def add_result(rid, finish):
    rows = load_dataset()
    for r in rows:
        if r["race_id"] == rid:
            r["result"] = finish
            save_dataset(rows)
            print(f"着順記録 {rid} -> {finish}")
            return
    print(f"該当無し: {rid}")

if __name__ == "__main__":
    a = sys.argv[1:]
    if not a:
        print("使い方:")
        print("  python collect.py 01 20260623")
        print("  python collect.py 01 20260623 1")
        print("  python collect.py result 20260623-01-01 1 2 3")
        sys.exit()
    if a[0] == "result":
        add_result(a[1], [int(x) for x in a[2:5]])
    else:
        jcd = int(a[0]); hd = a[1]
        only = int(a[2]) if len(a) > 2 else None
        collect(jcd, hd, only)
