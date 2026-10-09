# Data-QA-Testes-Automatizados-com-Great-Expectations

Para garantir que análises e modelos como o anterior funcionem corretamente, os dados de entrada precisam passar por pipelines de Garantia da Qualidade (Data Quality Assurance).

O Great Expectations é a biblioteca padrão em Python para declarar o que se espera dos dados através de "Expectations" (Expectativas) e validar datasets antes de alimentar bancos de dados ou modelos de ML.

Fluxo de Validação em Pipelines de Dados

[Fonte de Dados (CSV/SQL)] ➔ [Great Expectations Validation] ──(Sucesso)──➔ [Data Warehouse / Modelo ML]
                                           │
                                       (Falha)
                                           ↓
                              [Alerta Slack / Interrupção ETL]


# Implementação Prática em Python
Abaixo está um script pronto para validar um dataframe de dados industriais antes de ser ingerido no pipeline de produção:

import pandas as pd
import great_expectations as gx

# 1. Dados recebidos no pipeline (exemplo com erro intencional)
dados_producao = pd.DataFrame({
    'id_peca': [101, 102, 103, 104, 105],
    'diametro_mm': [15.01, 14.99, 15.02, 15.00, 18.50],  # 18.50 é um outlier gritante
    'status_inspecao': ['APROVADO', 'APROVADO', 'APROVADO', 'APROVADO', 'REJEITADO'],
    'codigo_operador': ['OP-01', 'OP-02', None, 'OP-01', 'OP-03'] # Contém nulo
})

# 2. Inicialização do contexto do Great Expectations
context = gx.get_context()
dataset_ge = gx.dataset.PandasDataset(dados_producao)

# 3. Definição das Regras de Qualidade (Expectations)

# A: O ID da peça deve ser sempre único e não nulo
dataset_ge.expect_column_values_to_be_unique(column='id_peca')
dataset_ge.expect_column_values_to_not_be_null(column='id_peca')

# B: O diâmetro deve estar estritamente dentro da faixa tolerada (14.90 mm a 15.10 mm)
dataset_ge.expect_column_values_to_be_between(
    column='diametro_mm', 
    min_value=14.90, 
    max_value=15.10
)

# C: O código do operador é obrigatório em todos os registros
dataset_ge.expect_column_values_to_not_be_null(column='codigo_operador')

# D: Os status só podem conter valores válidos da lista
dataset_ge.expect_column_values_to_be_in_set(
    column='status_inspecao', 
    value_set=['APROVADO', 'REJEITADO', 'RETRABALHO']
)

# 4. Execução dos Testes de Qualidade
resultado_validacao = dataset_ge.validate()

# 5. Tratamento de Sucesso ou Falha no Pipeline
if not resultado_validacao.success:
    print("❌ FALHA NA QUALIDADE DOS DADOS! O pipeline será interrompido.")
    
    # Exibir detalhes das falhas encontradas
    for resultado in resultado_validacao.results:
        if not resultado.success:
            print(f" -> Falha no teste: {resultado.expectation_config.expectation_type} "
                  f"na coluna '{resultado.expectation_config.kwargs.get('column')}'")
else:
    print("✅ Dados validados com sucesso! Prosseguindo com a carga de dados.")


# Como essa estrutura protege o sistema

Barreira de Entrada (Circuit Breaker): Se um sensor defeituoso gerar um valor irreal (como 18.50 mm), o teste expect_column_values_to_be_between falha e interrompe o pipeline antes que esse dado incorreto altere os KPIs da fábrica.

Documentação Automática: O Great Expectations gera relatórios visuais em HTML (Data Docs) descrevendo o estado de integridade de todas as tabelas.

Integração CI/CD: Esses scripts são executados em orquestradores como Apache Airflow ou ferramentas como dbt, garantindo que nenhum modelo receba dados corrompidos.
