# 🖨️ Sistema de Impressão 

Aplicativo desktop para Windows que imprime textos, senhas de banho, vales, fitas de cálculo e QR Codes de Wi-Fi em uma impressora térmica **Epson TM-T20X** (ESC/POS), com interface gráfica escura feita em `customtkinter`.

> Versão 2.1 · Criado por Warlen Kauã e Luan Delaflora

---

## ✨ Funcionalidades

| Módulo | O que faz |
|---|---|
| **Texto** | Imprime linhas de texto com cabeçalho opcional e tamanho de fonte à escolha (cada linha vira uma impressão separada). |
| **Banho** | Gera senhas únicas de 6 dígitos, imprime uma por vez e salva um CSV no formato de importação do GDC (pasta `CSV/`). |
| **Vale** | Imprime vales de combustível (um valor por linha) e registra cada um em `logs/vales.csv` para auditoria. |
| **Calculadora** | Fita de cálculo com `+ - * /` impressa no cupom (estilo calculadora de fita, sem precedência de operadores). |
| **Carta Frete** | Apura quebra de peso, desconto e saldo a receber, seguindo a planilha `CALCULO_FRETE_MANUAL.xlsx`. |
| **WiFi** | Imprime nome da rede, senha e QR Code de conexão automática (até 20 cópias). |
| **Config** | Escolhe a impressora instalada no Windows e as fontes padrão. As preferências ficam salvas em `config.json`. |

Outros detalhes:

- A interface não congela: as impressões rodam em uma thread separada, com progresso no rodapé (ex.: "Imprimindo 3 de 100...").
- Indicador de status da impressora no cabeçalho (verde = encontrada, vermelho = não encontrada).
- Valores monetários usam `Decimal`, evitando erros de arredondamento do `float`.
- Erros são gravados em `erros.log`, já que o `.pyw` não tem console.

---

## 📦 Requisitos

- **Windows** (usa `win32print` para falar com o spooler)
- **Python 3.9+**
- Impressora térmica compatível com ESC/POS (testado na **EPSON TM-T20X**, papel de 80 mm)
- Bibliotecas:

```bash
pip install customtkinter pywin32
```

---

## 🚀 Como executar

1. Clone o repositório:

   ```bash
   git clone https://github.com/<seu-usuario>/<seu-repositorio>.git
   cd <seu-repositorio>
   ```

2. Instale as dependências (veja acima).
3. Abra o arquivo **`app.pyw`** (duplo clique ou `pythonw app.pyw`).

Para ver erros no console durante o desenvolvimento, rode `python app.pyw`.

---

## 🗂️ Estrutura do projeto

```
├── app.pyw        # Ponto de entrada (abre a interface e registra erros em erros.log)
├── tela_ctk.py    # Interface gráfica (customtkinter) e ações de cada tela
├── main.py        # Comunicação ESC/POS, QR Code, parsing de valores e cálculos
├── config.json    # Preferências do usuário (gerado automaticamente)
├── CSV/           # Senhas de banho geradas (gerado automaticamente)
├── logs/
│   └── vales.csv  # Registro dos vales impressos (gerado automaticamente)
└── erros.log      # Log de erros (gerado automaticamente)
```

---

## ⚙️ Configuração

A impressora padrão é definida em `main.py`:

```python
NOME_IMPRESSORA = "EPSON TM-T20X Receipt6"
COLUNAS = 48  # 48 colunas para papel de 80 mm (use 32 para 58 mm)
```

Pela tela **Config** você também pode escolher outra impressora instalada no Windows sem editar o código. As escolhas são gravadas em `config.json`:

```json
{
  "impressora": "EPSON TM-T20X Receipt6",
  "cabecalho": "",
  "fonte_texto": "Grande (3x3)",
  "fonte_banho": "Gigante (8x8)",
  "wifi_titulo": "WIFI",
  "wifi_rede": "NOME_DA_REDE",
  "wifi_senha": "SENHA_DA_REDE"
}
```

Tamanhos de fonte disponíveis: `Normal`, `Média (2x2)`, `Grande (3x3)`, `Muito Grande (4x4)` e `Gigante (8x8)`.

---

## 🧮 Calculadora

Digite uma operação por linha. O operador vai no início; sem operador, o valor é somado.

```
100        → soma 100
-30        → subtrai 30
*2         → multiplica por 2
/4         → divide por 4
```

As operações são aplicadas de cima para baixo (estilo fita), a primeira linha deve ser um valor (sem `*` ou `/`) e o resultado é arredondado em 2 casas apenas no final.

---

## 🚚 Carta Frete

Baseada na planilha `CALCULO_FRETE_MANUAL.xlsx`:

- **Quebra** = peso líquido − peso da nota (negativo = faltou peso)
- **Isenção** = informada em **%** do peso da nota ou direto em **kg**
- **Valor da quebra** = se `|quebra|` > isenção: `(|quebra| − isenção) × valor por kg`; senão, 0
- **Valor saldo** = valor total − adiantamento 1 − adiantamento 2
- **Valor a receber** = valor saldo − valor da quebra

Aceita o formato brasileiro de números (`32.450`, `32450,5`, `7.948,40`).

> ⚠️ Como na planilha original, se o peso líquido ficar **acima** do peso da nota além da isenção, o excesso também é descontado como quebra. O app mostra um aviso nesse caso.

---

## 🚿 Senhas de banho

- Gera de 1 a 500 senhas únicas de 6 dígitos (módulo `secrets`).
- Imprime uma por vez, com o título "Banho de 7 minutos".
- Salva `CSV/senhas_DD-MM-AAAA_HH-MM-SS.csv` no formato do GDC: sem cabeçalho, separado por `;`, UTF-8 sem BOM:

  ```
  1;123456;0;7 min e 0 seg;1
  ```

- Se a impressão falhar no meio, o CSV guarda exatamente as senhas que chegaram a ser impressas.

---

## 📶 QR Code do Wi-Fi

Usa o padrão `WIFI:T:WPA;S:rede;P:senha;;` (com escape de caracteres especiais). Se a senha estiver vazia, o QR Code é gerado para rede aberta. Ao escanear com o celular, a conexão é feita automaticamente.

---

## 🛠️ Solução de problemas

- **"Impressora não encontrada"**: confira o nome exato em *Configurações e impressoras* do Windows ou use **Config → Atualizar lista**.
- **Acentos saindo errados**: o sistema usa a codepage 850; caracteres fora dela são impressos como `?`.
- **O sistema não abre**: veja o arquivo `erros.log` na pasta do programa.

---

## 🔒 Antes de publicar no GitHub

O `config.json` guarda a **senha do Wi-Fi** e os logs podem conter dados internos. Adicione ao `.gitignore`:

```
config.json
CSV/
logs/
erros.log
__pycache__/
```

---

## 🧰 Tecnologias

- [Python](https://www.python.org/)
- [customtkinter](https://github.com/TomSchimansky/CustomTkinter) — interface
- [pywin32](https://github.com/mhammond/pywin32) — impressão RAW via spooler do Windows
- ESC/POS — comandos de impressão e QR Code



