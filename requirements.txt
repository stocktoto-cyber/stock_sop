import streamlit as st
import pandas as pd
import numpy as np
import yfinance as yf
from datetime import datetime, timedelta

# ==========================================
# 0. Streamlit 頁面設定
# ==========================================
st.set_page_config(
    page_title="股市 SOP 戰情室",
    page_icon="📊",
    layout="wide"
)

# ==========================================
# 1. 側邊欄與輸入設定 (取代原本的 Markdown Param)
# ==========================================
st.sidebar.header("🚀 股票設定")
target_stocks_input = st.sidebar.text_input("輸入股號 (逗號分隔)", value="006208")

st.sidebar.header("🗓️ 模式選擇")
quick_range_options = [
    "⚡ 即時戰情 (Today)", "近 1 年", "近 3 年", 
    "2024 (AI爆發)", "2023 (盤整復甦)", 
    "2022 (升息/空頭)", "2021 (航運/大牛)", "自訂 (Custom)"
]
quick_range_picker = st.sidebar.selectbox("選擇區間", options=quick_range_options, index=1)

# 手動日期 (僅在「自訂」模式生效)
st.sidebar.markdown("---")
is_custom = quick_range_picker == "自訂 (Custom)"
start_date_picker = st.sidebar.date_input("開始日期", value=datetime(2024, 1, 1), disabled=not is_custom)
use_today_as_end = st.sidebar.checkbox("結束日期使用今天?", value=True, disabled=not is_custom)
end_date_picker = st.sidebar.date_input("結束日期", value=datetime(2025, 1, 1), disabled=(not is_custom or use_today_as_end))

run_btn = st.sidebar.button("🚀 啟動分析", type="primary")

# ==========================================
# 2. 核心邏輯 (保持不變)
# ==========================================

# 設定台灣時間
now_tw = datetime.utcnow() + timedelta(hours=8)
today = now_tw
today_str = today.strftime('%Y-%m-%d')

def parse_tickers(input_str):
    stocks = []
    for s in input_str.split(","):
        s = s.strip()
        if not s: continue
        if ".TW" not in s and s[0].isdigit():
            stocks.append(s + ".TW")
        else:
            stocks.append(s)
    return stocks

def get_pattern_trend(df_week):
    if len(df_week) < 5: return "無明顯型態"
    p_close = df_week['Close'].values
    p_low = df_week['Low'].values
    p_high = df_week['High'].values

    if p_low[-1] > p_low[-2] and p_low[-2] > p_low[-3]:
        return "📈 底底高 (多方)"
    elif p_high[-1] < p_high[-2] and p_high[-2] < p_high[-3]:
        return "📉 峰峰低 (空方)"
    elif p_close[-1] > p_close[-2]:
        return "↗️ 反彈/上漲"
    else:
        return "↘️ 拉回/整理"

# 加入快取以提升 Streamlit 效能
@st.cache_data(ttl=300)
def fetch_stock_history(stock_id, start_date, end_date):
    ticker = yf.Ticker(stock_id)
    # yfinance 需要 datetime 或 str，這裡做簡單轉換確保格式
    d_start = pd.to_datetime(start_date) - timedelta(days=365)
    # 如果 end_date 是 datetime.date 物件，轉為 datetime
    if isinstance(end_date,  datetime):
        d_end = end_date
    else:
        d_end = pd.to_datetime(end_date)
    
    d_end = d_end + timedelta(days=90)
    
    hist = ticker.history(start=d_start, end=d_end, interval="1wk")
    hist.index = hist.index.tz_localize(None)
    return hist

def analyze_stock(stock_id, start_date, end_date, realtime_mode=False):
    try:
        # 使用封裝後的資料抓取函數
        hist = fetch_stock_history(stock_id, start_date, end_date)

        if len(hist) < 30: return []

        if realtime_mode:
            target_weeks = [hist.index[-1]]
        else:
            mask = (hist.index >= pd.to_datetime(start_date)) & (hist.index <= pd.to_datetime(end_date))
            target_weeks = hist[mask].index

        signals = []
        last_idx_time = hist.index[-1]

        for date in target_weeks:
            curr_idx = hist.index.get_loc(date)
            past_data = hist.iloc[:curr_idx+1].copy()

            # --- 技術指標 ---
            past_data['MA20'] = past_data['Close'].rolling(window=20).mean()
            past_data['STD'] = past_data['Close'].rolling(window=20).std()
            past_data['Upper'] = past_data['MA20'] + (2 * past_data['STD'])
            past_data['Lower'] = past_data['MA20'] - (2 * past_data['STD'])

            curr = past_data.iloc[-1]
            ma20 = curr['MA20']
            lower = curr['Lower']
            upper = curr['Upper']
            close = curr['Close']
            volume = curr['Volume']
            open_p = curr['Open']
            high = curr['High']

            # --- 量能與預估量 ---
            avg_vol = past_data['Volume'].iloc[-2:-7:-1].mean()

            # 判斷是否為最新週
            is_latest_week = (date == last_idx_time) and ((today - date).days < 7)

            if is_latest_week and realtime_mode:
                display_date = today_str
                weekday = today.weekday()
                proj_factor = 5 / max(weekday + 1, 0.5) if weekday < 5 else 1
                final_vol = int(volume * proj_factor)
                vol_note = "(預估)"
            else:
                display_date = date.strftime('%Y-%m-%d')
                final_vol = int(volume)
                vol_note = ""

            vol_msg = "🔥爆量" if final_vol > (avg_vol * 1.5) else ("💧量縮" if final_vol < (avg_vol * 0.7) else "溫和")

            # --- 欄位計算 ---
            ma_status = "🔴站上" if close > ma20 else "🟢跌破"

            if close > upper: bb_pos = "🚀 突破上緣"
            elif close > ma20: bb_pos = "🔴 多方區"
            elif close < lower: bb_pos = "🟢 跌破下緣"
            else: bb_pos = "📉 空方區"

            pattern_trend = get_pattern_trend(past_data)

            # --- 防詐騙 & 空間 ---
            is_black = close < open_p
            upper_shadow = high - max(open_p, close)
            body = abs(close - open_p)
            is_shooting_star = upper_shadow > (body * 1.5) and upper_shadow > (close * 0.01)
            rebound_space = ((ma20 - close) / close) * 100

            # --- SOP 建議 ---
            action = "觀察"
            if close > ma20:
                if "爆量" in vol_msg:
                    if is_black or is_shooting_star:
                        action = "⚠️ 爆量假突破"
                    else:
                        action = "★ 強力買進"
                elif "多方" in pattern_trend:
                    action = "★ 趨勢多方"
                else:
                    action = "續抱/觀望"
            elif close < ma20:
                if close < lower and "爆量" in vol_msg:
                    action = "⚡ 低檔爆量 (肉多)" if rebound_space > 10 else "⚡ 低檔爆量 (肉少)"
                else:
                    action = "⚠️ 賣出/避開"

            # --- 績效驗證 ---
            if not realtime_mode:
                future_idx = curr_idx + 4
                if future_idx < len(hist):
                    exit_price = hist.iloc[future_idx]['Close']
                    ret_pct = ((exit_price - close) / close) * 100
                else:
                    ret_pct = np.nan
            else:
                ret_pct = np.nan

            # --- 存入結果 ---
            if realtime_mode or ("買進" in action or "低檔" in action or "假突破" in action):
                signals.append({
                    '代號': stock_id.replace('.TW', ''),
                    '日期': display_date,
                    '股價': close,
                    'MA20': ma20,
                    '均線狀態': ma_status,
                    '布林位置': bb_pos,
                    '型態趨勢': pattern_trend,
                    '成交量(張)': int(volume / 1000),
                    '預估週量(張)': int(final_vol / 1000),
                    '量能訊號': f"{vol_msg}{vol_note}",
                    '建議操作': action,
                    '績效(%)': ret_pct
                })

        return signals

    except Exception as e:
        print(f"Error {stock_id}: {e}")
        return []

# ==========================================
# 3. 表格美化 (保持不變)
# ==========================================
def style_results(df, realtime=False):
    def color_action(val):
        if '強力買進' in str(val): return 'color: white; background-color: #d62728; font-weight: bold'
        if '肉多' in str(val) or '搶反彈' in str(val): return 'color: black; background-color: #ffcc00; font-weight: bold'
        if '肉少' in str(val): return 'color: black; background-color: #fffacd'
        if '假突破' in str(val): return 'color: white; background-color: #800080'
        if '賣出' in str(val): return 'color: white; background-color: #2ca02c'
        return ''

    def color_return(val):
        if pd.isna(val): return ''
        if val > 0: return 'color: red; font-weight: bold'
        if val < 0: return 'color: green; font-weight: bold'
        return 'color: gray'

    cols = ['代號', '日期', '股價', 'MA20', '均線狀態', '布林位置', '型態趨勢', '成交量(張)', '預估週量(張)', '量能訊號', '建議操作']
    if not realtime:
        cols.append('績效(%)')

    format_dict = {
        '股價':'{:.1f}',
        'MA20':'{:.1f}',
        '成交量(張)':'{:,.0f}',
        '預估週量(張)':'{:,.0f}',
        '績效(%)':'{:.2f}%'
    }

    # 為了相容性，這裡使用 map 替代部分環境已棄用的 applymap，若環境較舊可改回 applymap
    styler = df[cols].style.format(format_dict)\
              .map(color_action, subset=['建議操作'])

    if not realtime:
        styler = styler.map(color_return, subset=['績效(%)'])

    return styler.set_properties(**{'text-align': 'center'})\
                 .set_table_styles([{'selector': 'th', 'props': [('background-color', '#404040'), ('color', 'white')]}])

# ==========================================
# 4. 主程式執行邏輯
# ==========================================
st.title("📊 股市 SOP 戰情室 (Streamlit 版)")

if run_btn:
    # --- 日期區間邏輯 ---
    scenario_name = quick_range_picker
    is_realtime_mode = False
    final_start = None
    final_end = None

    if "即時戰情" in quick_range_picker:
        is_realtime_mode = True
        final_start = today - timedelta(days=365)
        final_end = today
    elif quick_range_picker == "近 1 年":
        final_end = today
        final_start = today - timedelta(days=365)
    elif quick_range_picker == "近 3 年":
        final_end = today
        final_start = today - timedelta(days=365*3)
    elif "2024" in quick_range_picker:
        final_start = datetime(2024, 1, 1)
        final_end = today
    elif "2023" in quick_range_picker:
        final_start = datetime(2023, 1, 1)
        final_end = datetime(2023, 12, 31)
    elif "2022" in quick_range_picker:
        final_start = datetime(2022, 1, 1)
        final_end = datetime(2022, 12, 31)
    elif "2021" in quick_range_picker:
        final_start = datetime(2021, 1, 1)
        final_end = datetime(2021, 12, 31)
    else: # 自訂
        scenario_name = "自訂區間"
        final_start = pd.to_datetime(start_date_picker)
        if use_today_as_end:
            final_end = today
        else:
            final_end = pd.to_datetime(end_date_picker)

    # --- 顯示狀態 ---
    status_text = ""
    if is_realtime_mode:
        status_text = f"⚡ 即時戰情模式啟動 | 時間：{today_str} (UTC+8)"
    else:
        status_text = f"🎬 回測劇本：{scenario_name} | 📅 掃描區間：{final_start.strftime('%Y-%m-%d')} 至 {final_end.strftime('%Y-%m-%d')}"
    
    st.info(status_text)
    
    # --- 開始分析 ---
    target_stocks = parse_tickers(target_stocks_input)
    
    with st.spinner('分析中，請稍候...'):
        all_signals = []
        for stock in target_stocks:
            sigs = analyze_stock(stock, final_start, final_end, is_realtime_mode)
            all_signals.extend(sigs)

        if all_signals:
            df = pd.DataFrame(all_signals)
            df = df.sort_values(by=['日期', '代號'], ascending=[False, True])

            # 統計顯示
            if not is_realtime_mode:
                valid = df.dropna(subset=['績效(%)'])
                if len(valid) > 0:
                    wins = len(valid[valid['績效(%)'] > 0])
                    win_rate = (wins / len(valid)) * 100
                    st.metric(label="🏆 回測勝率", value=f"{win_rate:.1f}%", delta=f"交易次數: {len(valid)}")

            # 顯示表格
            st.dataframe(style_results(df, is_realtime_mode), height=600, use_container_width=True)
            
        else:
            st.warning("🍃 無符合條件的訊號。")

else:
    st.write("👈 請在左側設定後點擊「啟動分析」")