# Projeto-Mapa-das-UPAS-SES
# MAPA DAS UPAS DO DF - VERSÃO FINAL COM LOCALIZAÇÃO POR CLIQUE E CEP

import streamlit as st
import geopandas as gpd
import pandas as pd
import folium
from streamlit_folium import st_folium
from folium.plugins import LocateControl, MousePosition
import os

# Configurações iniciais
debug = False
st.set_page_config(layout="wide")
st.markdown("<h1 style='font-size: 28px;'>Unidades de Pronto Atendimento (UPA) - Mapa Interativo</h1>", unsafe_allow_html=True)

# Caminho base
BASE_PATH = "C:/Users/07045630107/Desktop/MAPA- UPAs"

@st.cache_data
def load_data():
    upas = gpd.read_file(os.path.join(BASE_PATH, "UPA_atual_oct2023.shp")).to_crs(epsg=4326)
    ceps = gpd.read_file(os.path.join(BASE_PATH, "CEPS_2024.shp")).to_crs(epsg=4326)
    ras = gpd.read_file(os.path.join(BASE_PATH, "regioes_administrativas.shp")).to_crs(epsg=4326)

    cep_col = [col for col in ceps.columns if "cep" in col.lower()][0]
    col_log_final = next((col for col in ceps.columns if "LOG_NO_ABR" in col), None)

    ceps["CEP_LIMPO"] = ceps[cep_col].apply(lambda x: str(int(float(x))) if pd.notnull(x) else "")
    ceps["ENDERECO_BUSCA"] = ceps[col_log_final].fillna("").astype(str).str.strip().str.upper()

    return upas, ceps, ras

# Carrega dados
upas, ceps, ras = load_data()

# Ícones
icon_pediatrico = os.path.join(BASE_PATH, "icone_upa_rosa.png")
icon_geral = os.path.join(BASE_PATH, "icone_UPA.png")
pin_icon = folium.CustomIcon(os.path.join(BASE_PATH, "pin azul.png"), icon_size=(60, 60))

# Sidebar
with st.sidebar:
    st.markdown("<div style='padding:10px; border: 3px solid #007b5e; border-radius: 10px; font-weight: bold; font-size: 18px; background-color: #e6f2f0;'>UPAs do DF</div>", unsafe_allow_html=True)
    st.markdown("<div style='margin-top:15px; font-size: 16px;'>Buscar por CEP ou Endereço:</div>", unsafe_allow_html=True)
    todas_opcoes = sorted(set(list(ceps["CEP_LIMPO"]) + list(ceps["ENDERECO_BUSCA"])))
    todas_opcoes = [op for op in todas_opcoes if op and not any(char.isdigit() for char in op if char not in "0123456789")] + [op for op in todas_opcoes if op.isnumeric()]
    busca_input = st.selectbox("", options=todas_opcoes, index=None, placeholder="Digite o CEP ou endereço...")

# Mapa base:
m = folium.Map(location=[-15.8, -47.8], zoom_start=10)
LocateControl(auto_start=False, keepCurrentZoomLevel=True).add_to(m)
MousePosition().add_to(m)

# Camada RA:
folium.GeoJson(
    ras,
    name="Regiões Administrativas",
    style_function=lambda feature: {
        'color': 'black',         # contorno preto
        'weight': 2,              # espessura da linha
        'fill': False,            # sem preenchimento
        'fillOpacity': 0          # reforço visual para garantir transparência
    },
    tooltip=folium.GeoJsonTooltip(fields=["ra_nome"], aliases=["RA:"]) #serve para mostrar o nome da RA ao passar o mouse
).add_to(m)

# Função para adicionar as 3 UPAs mais próximas de um ponto:
def mostrar_upas_proximas(ponto_geom):
    upas["dist_km"] = upas.geometry.distance(ponto_geom) * 111
    pedi = upas[upas["TP_UNIDADE"].str.lower().str.strip() == "sim"]
    n_pedi = upas[upas["TP_UNIDADE"].str.lower().str.strip() != "sim"]
    mais_proximas = pd.concat([n_pedi.nsmallest(2, "dist_km"), pedi.nsmallest(1, "dist_km")])

    for _, row in mais_proximas.iterrows():
        pediatrico = str(row.get('TP_UNIDADE', '')).strip().lower() == 'sim'
        icone = folium.CustomIcon(icon_pediatrico if pediatrico else icon_geral, icon_size=(50, 50))
        tooltip = f"""
        <b>UPA:</b> {row['NOME_FANTA']}<br>
        <b>Endereço:</b> {row['LOGRADOURO']}, {row.get('NUMERO', '')}, {row['BAIRRO']}<br>
        <b>CEP:</b> {row.get('CEP', 'Não informado')}<br>
        <b>RA:</b> {row.get('Reg_Sa__de', 'Não informado')}<br>
        <b>Tipo:</b> {'Inclui atendimento Pediátrico' if pediatrico else 'Não inclui atendimento Pediátrico'}
        """
        folium.Marker([row.geometry.y, row.geometry.x], tooltip=tooltip, icon=icone).add_to(m)

    return mais_proximas

# Lógica para busca por CEP ou endereço:
if busca_input:
    resultado = ceps[(ceps["CEP_LIMPO"] == busca_input) | (ceps["ENDERECO_BUSCA"] == busca_input)]
    if not resultado.empty:
        ponto = resultado.iloc[0].geometry
        folium.Marker([ponto.y, ponto.x], tooltip=f"Local: {busca_input}", icon=pin_icon).add_to(m)
        m.location = [ponto.y, ponto.x]
        m.zoom_start = 13

        mais_proximas = mostrar_upas_proximas(ponto)

        with st.sidebar:
            st.markdown("### UPAs mais próximas:")
            selecao = st.radio("Selecione uma UPA:", mais_proximas["NOME_FANTA"].tolist(), index=0)

            selecionada = mais_proximas[mais_proximas["NOME_FANTA"] == selecao].iloc[0]
            st.markdown("### ℹ Detalhes da UPA Selecionada")
            st.markdown(f"""
            <div style='font-size:16px;'>
            <b>UPA:</b> {selecionada['NOME_FANTA']}<br>
            <b>Endereço:</b> {selecionada['LOGRADOURO']} Nº {selecionada.get('NUMERO', '')}<br>
            <b>Bairro:</b> {selecionada['BAIRRO']}<br>
            <b>CEP:</b> {selecionada['CEP']}<br>
            <b>Região Administrativa:</b> {selecionada.get('Reg_Sa__de', 'Não informado')}<br>
            <b>Tipo:</b> {'Inclui atendimento Pediátrico' if str(selecionada.get('TP_UNIDADE', '')).strip().lower() == 'sim' else 'Não inclui atendimento Pediátrico'}
            </div>
            """, unsafe_allow_html=True)
    else:
        st.sidebar.error("CEP ou endereço não encontrado. Por favor, tente novamente.")
else:
    # Se nenhum CEP/endereço, mostrar todas as UPAs inicialmente:
    for _, row in upas.iterrows():
        pediatrico = str(row.get('TP_UNIDADE', '')).strip().lower() == 'sim'
        icone = folium.CustomIcon(icon_pediatrico if pediatrico else icon_geral, icon_size=(60, 60))
        tooltip = f"""
        <b>UPA:</b> {row['NOME_FANTA']}<br>
        <b>Endereço:</b> {row['LOGRADOURO']}, {row.get('NUMERO', '')}, {row['BAIRRO']}<br>
        <b>CEP:</b> {row.get('CEP', 'Não informado')}<br>
        <b>RA:</b> {row.get('Reg_Sa__de', 'Não informado')}<br>
        <b>Tipo:</b> {'Inclui atendimento Pediátrico' if pediatrico else 'Não inclui atendimento Pediátrico'}
        """
        folium.Marker([row.geometry.y, row.geometry.x], tooltip=tooltip, icon=icone).add_to(m)
  # Função para cliques no mapa
map_data = st_folium(m, width=1100, height=800)

if map_data.get("last_clicked"):
    ponto = gpd.GeoSeries(gpd.points_from_xy(
        [map_data["last_clicked"]["lng"]],
        [map_data["last_clicked"]["lat"]]
    ), crs="EPSG:4326").iloc[0]

    # Adiciona ao mesmo mapa original (sem recriar o mapa)
    folium.Marker([ponto.y, ponto.x], tooltip="Local escolhido", icon=pin_icon).add_to(m)
    mostrar_upas_proximas(ponto)
