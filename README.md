# FIN.ROBOT
it's a simple robot my friend asked for.
import streamlit as st
import yfinance as yf
import pandas as pd
import plotly.graph_objects as go

# =========================
# CONFIGURAÇÃO DA PÁGINA
# =========================
st.set_page_config(
    page_title="Robô Trading PRO",
    page_icon="📈",
    layout="wide"
)

st.title("🤖 Robô Trading PRO")

# =========================
# ATIVOS DISPONÍVEIS
# =========================
ativos = {
    "OURO": "GC=F",
    "BITCOIN": "BTC-USD",
    "ETHEREUM": "ETH-USD",
    "NASDAQ": "^IXIC"
}

# =========================
# TIMEFRAMES
# =========================
timeframes = {
    "5 minutos": ("5d", "5m"),
    "15 minutos": ("15d", "15m"),
    "1 hora": ("30d", "60m"),
    "4 horas": ("60d", "120m"),
    "Diário": ("6mo", "1d")
}

# =========================
# SELEÇÃO DO USUÁRIO
# =========================
col1, col2 = st.columns(2)

with col1:
    ativo_escolhido = st.selectbox("Escolha o ativo", list(ativos.keys()))

with col2:
    timeframe_escolhido = st.selectbox("Escolha o timeframe", list(timeframes.keys()))

periodo, intervalo = timeframes[timeframe_escolhido]

# =========================
# BAIXAR DADOS
# =========================
@st.cache_data
def carregar_dados(ticker, periodo, intervalo):
    dados = yf.download(
        ticker,
        period=periodo,
        interval=intervalo,
        auto_adjust=True,
        progress=False
    )
    return dados

dados = carregar_dados(ativos[ativo_escolhido], periodo, intervalo)

# se falhar o download
if dados.empty:
    st.error("Não foi possível carregar dados do ativo.")
    st.stop()

# =========================
# AJUSTAR TIMEZONE (Brasil)
# =========================
if dados.index.tz is not None:
    dados.index = dados.index.tz_convert("America/Sao_Paulo")

# =========================
# INDICADORES
# =========================
dados["MM9"] = dados["Close"].rolling(9).mean()
dados["MM21"] = dados["Close"].rolling(21).mean()

# =========================
# GRÁFICO CANDLESTICK
# =========================
fig = go.Figure()

fig.add_trace(go.Candlestick(
    x=dados.index,
    open=dados['Open'],
    high=dados['High'],
    low=dados['Low'],
    close=dados['Close'],
    name="Candles"
))

fig.add_trace(go.Scatter(
    x=dados.index,
    y=dados["MM9"],
    mode="lines",
    name="MM9"
))

fig.add_trace(go.Scatter(
    x=dados.index,
    y=dados["MM21"],
    mode="lines",
    name="MM21"
))

fig.update_layout(
    xaxis_rangeslider_visible=False,
    height=600
)

# =========================
# FORMATAÇÃO PROFISSIONAL DO TEMPO
# =========================
fig.update_xaxes(
    tickformatstops=[
        dict(dtickrange=[None, 3600000], value="%H:%M"),          # minutos
        dict(dtickrange=[3600000, 86400000], value="%d/%m %H:%M"),# horas
        dict(dtickrange=[86400000, None], value="%d/%m/%Y"),      # dias
    ]
)

st.plotly_chart(fig, use_container_width=True)

# =========================
# INFORMAÇÕES RÁPIDAS
# =========================
st.subheader("📊 Informações do Mercado")

col1, col2, col3 = st.columns(3)

col1.metric("Preço Atual", f"${float(dados['Close'].iloc[-1]):.2f}")
col2.metric("Máxima", f"${float(dados['High'].iloc[-1]):.2f}")
col3.metric("Mínima", f"${float(dados['Low'].iloc[-1]):.2f}")

# =========================
# TABELA DE DADOS
# =========================
st.subheader("📋 Dados Recentes")
st.dataframe(dados.tail(10))
