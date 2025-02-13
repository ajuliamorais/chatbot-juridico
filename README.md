# chatbot-juridicoimport spacy
import re
import streamlit as st
import pdfplumber
from docx import Document

# Carrega o modelo de NLP em português
nlp = spacy.load("pt_core_news_lg")

def extrair_texto_arquivo(uploaded_file):
    if uploaded_file.type == "text/plain":
        return uploaded_file.read().decode("utf-8")
    elif uploaded_file.type == "application/pdf":
        with pdfplumber.open(uploaded_file) as pdf:
            return "\n".join([page.extract_text() for page in pdf.pages])
    elif uploaded_file.type == "application/vnd.openxmlformats-officedocument.wordprocessingml.document":
        doc = Document(uploaded_file)
        return "\n".join([para.text for para in doc.paragraphs])
    else:
        raise ValueError("Formato de arquivo não suportado")

def analisar_contrato(texto):
    doc = nlp(texto)
    
    resultados = {
        "velocidade_contratada": None,
        "minimo_garantido": None,
        "conformidade_velocidade": None,
        "prazo_fidelidade": re.search(r"\d+\s+(meses|anos)", texto),
        "multa_cancelamento": re.search(r"multa\s+de\s+(\d+%)", texto, re.IGNORECASE),
        "clausulas_abusivas": [],
        "politica_privacidade": "LGPD" in texto
    }

    # Análise de velocidade
    match_velocidade = re.search(r"(\d+)\s*Mbps", texto, re.IGNORECASE)
    if match_velocidade:
        velocidade = int(match_velocidade.group(1))
        resultados["velocidade_contratada"] = velocidade
        
        # Verificação ANATEL (70% mínimo)
        minimo_regulatorio = velocidade * 0.7
        resultados["minimo_garantido"] = minimo_regulatorio
        resultados["conformidade_velocidade"] = "Conforme" if minimo_regulatorio <= velocidade else "Inconforme"

    # Cláusulas abusivas
    termos_abusivos = ["unilateralmente", "alteração de preço", "proibido cancelar"]
    for sent in doc.sents:
        if any(palavra in sent.text.lower() for palavra in termos_abusivos):
            resultados["clausulas_abusivas"].append(sent.text)

    return resultados

# Interface Streamlit
st.title("🤖 Analisador de Contratos de Internet")
uploaded_file = st.file_uploader("Carregue seu contrato (PDF, DOCX ou TXT)", type=["pdf", "docx", "txt"])

if uploaded_file:
    texto = extrair_texto_arquivo(uploaded_file)
    analise = analisar_contrato(texto)
    
    st.subheader("📋 Resultados da Análise")
    
    col1, col2 = st.columns(2)
    
    with col1:
        st.metric("Velocidade Contratada", f"{analise['velocidade_contratada']} Mbps" if analise['velocidade_contratada'] else "Não encontrada")
        st.metric("Mínimo Garantido", f"{analise['minimo_garantido']:.1f} Mbps" if analise['minimo_garantido'] else "N/A")
        st.metric("Conformidade ANATEL", analise['conformidade_velocidade'] or "N/A")
    
    with col2:
        st.write("**Cláusulas Potencialmente Abusivas:**")
        for clausula in analise['clausulas_abusivas']:
            st.error(f"⚠️ {clausula}")
        
        if analise['politica_privacidade']:
            st.success("✅ Política de Privacidade (LGPD) Detectada")
        else:
            st.warning("⚠️ Menção à LGPD não encontrada")

    st.divider()
    st.subheader("Texto Extraído do Contrato")
    st.text(texto[:1000] + "...")  # Exibe apenas os primeiros 1000 caracteres
