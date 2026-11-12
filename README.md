# 🖨️ Sistema de Impressão 

Aplicativo desktop para Windows que centraliza as impressões do dia a dia de um posto de combustível numa impressora térmica **Epson TM-T20X** e se integra ao **GDC (Gerenciador de Duchas Companytec)** para cadastrar automaticamente as senhas de banho.

![Python](https://img.shields.io/badge/Python-3.9%2B-3776AB?logo=python&logoColor=white)
![Plataforma](https://img.shields.io/badge/Windows-10%20%7C%2011-0078D6?logo=windows&logoColor=white)
![Interface](https://img.shields.io/badge/UI-customtkinter-C51F26)

<!-- Adicione aqui um print da tela principal, por exemplo: ![Tela principal](docs/tela.png) -->

## Sumário

- [Funcionalidades](#funcionalidades)
- [Como funciona a senha de banho](#como-funciona-a-senha-de-banho)
- [Requisitos](#requisitos)
- [Instalação e uso](#instalação-e-uso)
- [Configuração](#configuração)
- [Estrutura do projeto](#estrutura-do-projeto)
- [Arquivos gerados](#arquivos-gerados)
- [Gerar o executável (.exe)](#gerar-o-executável-exe)
- [Solução de problemas](#solução-de-problemas)
- [Privacidade e segurança](#privacidade-e-segurança)
- [Autor e licença](#autor-e-licença)

## Funcionalidades

| Aba | O que faz |
|---|---|
| **Início (Texto)** | Imprime texto livre, uma impressão por linha, com cabeçalho e tamanho de fonte à escolha (Normal, Média 2x2, Grande 3x3, Muito Grande 4x4 e Gigante 8x8). Limite de 1000 caracteres. |
| **Banho** | Gera senhas únicas de 6 dígitos, cadastra no GDC pela rede e imprime somente as que o GDC confirmar. Também salva um CSV no formato de importação do GDC. |
| **Vale** | Imprime vales de combustível (um por linha). Valida valores maiores que zero, pede confirmação com o total e registra cada vale em `logs/vales.csv`. |
| **Calculadora** | Calculadora em "fita": `+ - * /` no início de cada linha (sem sinal, soma) e impressão da fita no cupom. |
| **Carta Frete** | Calcula o valor do frete, o saldo e, opcionalmente, o desconto por quebra de peso. Veja [Carta Frete](#carta-frete). |
| **WiFi** | Imprime o nome da rede, a senha e um **QR Code** de conexão (`WIFI:T:WPA;S:…;P:…;;`), com opção de várias cópias. |
| **Config** | Escolhe a impressora, as fontes padrão, o IP e a porta do GDC, a pausa entre senhas e testa a conexão. Tudo fica salvo em `config.json`. |
| **Sobre** | Nome do projeto, versão, autor e comentários. |

Outros destaques:

- Impressão em **segundo plano**: a janela não congela e o progresso aparece na barra de status.
- Erros de impressora com **mensagens claras** (desligada, sem papel, nome errado), sem travar o programa.
- Valores monetários calculados com `Decimal` (sem erros de arredondamento de `float`).
- Senhas geradas com o módulo `secrets` (não previsível).

### Carta Frete

```
valor do frete = (peso de saída em toneladas) × (valor por tonelada)     → fechado em centavos
saldo          = valor do frete − adiantamentos
```

**Quebra de peso (opcional)** — só entra se o *peso de chegada* for informado:

```
quebra    = peso de chegada − peso de saída
isenção   = informada em % do peso de saída ou direto em kg
desconto  = (|quebra| − isenção) × valor por kg da quebra     (se |quebra| > isenção)
a receber = saldo − desconto
```

> A regra da quebra segue a planilha de cálculo manual usada no posto, que aplica o valor absoluto: um peso de chegada *acima* do de saída além da isenção também é descontado. A tela exibe um aviso nesse caso.

Nos campos numéricos, use **vírgula** para decimais. `32.050` é lido como *trinta e dois mil e cinquenta*.

## Como funciona a senha de banho

Para cada senha, o programa:

1. **Gera** uma senha de 6 dígitos (100000–999999) que ainda não foi usada.
2. **Cadastra no GDC** por TCP. A conexão fica aberta durante o lote e é reaberta automaticamente se o GDC fechar, der timeout ou responder algo inesperado.
3. **Imprime só se o GDC confirmar** (`{ST&CD}`), em lotes de até 10 senhas por trabalho de impressão. O cadastro e a impressão acontecem **ao mesmo tempo**.
4. **Registra** a senha nos arquivos de controle.

Garantias e proteções:

- Senha **não confirmada pelo GDC não é impressa**; ao final, o programa lista o motivo de cada falha.
- Senha com timeout **não é reenviada automaticamente** (o GDC pode tê-la gravado mesmo sem a confirmação chegar).
- Após **3 falhas de rede seguidas**, o processo é interrompido (GDC desligado ou fora da rede).
- Se a impressora falhar depois de o GDC confirmar, a senha fica **pendente**: o botão **Imprimir pendentes** reimprime sem recadastrar.
- Nunca repete senha já usada (`senhas_gravadas.txt` + `impresso.txt`).
- O interruptor **"Cadastrar no GDC antes de imprimir"** permite apenas imprimir (por exemplo, com a rede fora); nesse caso, importe o CSV gerado no GDC Manager.
- Ao final, o aviso mostra o **tempo total** e a média por senha, para ajudar a achar gargalos.

**CSV gerado** (formato de importação do GDC, sem cabeçalho, separado por `;`, UTF-8 sem BOM):

```
1;643890;0;7 min e 0 seg;1
2;100234;0;7 min e 0 seg;1
```

Colunas: número, senha, ducha (`0`), duração e validade (`1` uso).

**Protocolo.** Cada comando é um quadro `{` + dados + `&` + checksum (2 dígitos hexadecimais) + `}`; a resposta de sucesso é `{ST&CD}`. A duração (7 min 0 s), a ducha e a validade ficam em constantes no topo de `gdc.py`. Documentação oficial: [kit de desenvolvimento da Companytec](https://github.com/companytec/companytec_kit_desenvolvimento).

## Requisitos

- **Windows** 10 ou 11 (usa a API de impressão do Windows via `pywin32`)
- **Python 3.9 ou superior** (desenvolvido com 3.14)
- Impressora térmica **Epson TM-T20X** (ESC/POS, 80 mm) instalada no Windows
- GDC Companytec acessível na rede local (opcional, só para o cadastro automático das senhas)

## Instalação e uso

```bash
git clone https://github.com/<seu-usuario>/<seu-repositorio>.git
cd <seu-repositorio>

pip install -r requirements.txt

pythonw app.pyw        # abre sem janela de console
# ou: python app.pyw
```

Na primeira execução, abra a aba **Config** e informe:

1. o **nome da impressora** (como aparece no Windows, ex.: `EPSON TM-T20X Receipt6`);
2. o **IP e a porta do GDC** (padrão `10.1.1.199:888`) e use **Testar GDC**;
3. clique em **Salvar e testar**.

## Configuração

As preferências ficam em `config.json`, ao lado do programa, e são criadas automaticamente:

| Chave | Descrição | Padrão |
|---|---|---|
| `impressora` | Nome da impressora no Windows | `EPSON TM-T20X Receipt6` |
| `cabecalho` | Cabeçalho da aba Texto | `Sistema de impressão` |
| `fonte_texto` / `fonte_banho` | Tamanho de fonte padrão | `Média (2x2)` |
| `wifi_titulo` / `wifi_rede` / `wifi_senha` | Dados do QR Code do WiFi | veja `CONFIG_PADRAO` |
| `gdc_ip` / `gdc_porta` | Endereço do GDC | `10.1.1.199` / `888` |
| `gdc_ativo` | Cadastrar no GDC antes de imprimir | `true` |
| `pausa_gdc` | Pausa (s) entre senhas enviadas ao GDC | `0.1` |

Parâmetros de desempenho e limites ficam em constantes no topo dos arquivos: `LOTE_IMPRESSAO`, `MAX_SENHAS`, `MAX_FALHAS_SEGUIDAS` (`tela_ctk.py`) e `TIMEOUT_CONEXAO`, `TIMEOUT_RESPOSTA`, `DURACAO_MINUTOS` (`gdc.py`).

## Estrutura do projeto

```
.
├── app.pyw            # ponto de entrada (log de erros e tratamento de falha ao iniciar)
├── tela_ctk.py        # interface (customtkinter), fluxos de impressão e configuração
├── main.py            # impressora ESC/POS, QR Code, cálculo de carta frete e utilitários de valores
├── gdc.py             # cliente do GDC (TCP, conexão persistente, protocolo)
├── icone.ico          # ícone do programa
├── compilar.bat       # gera o .exe com PyInstaller
├── requirements.txt
└── README.md
```

## Arquivos gerados

Criados ao lado do programa (ou do `.exe`):

| Arquivo | Conteúdo |
|---|---|
| `config.json` | Preferências do usuário |
| `CSV/senhas_<data>.csv` | Senhas impressas, no formato de importação do GDC |
| `senhas_gravadas.txt` | Senhas confirmadas pelo GDC |
| `impresso.txt` | Senhas já impressas (a diferença entre os dois são as *pendentes*) |
| `logs/vales.csv` | Registro de cada vale impresso |
| `erros.log` | Erros e avisos do programa |

> ⚠️ **Não apague** `senhas_gravadas.txt` e `impresso.txt`: eles evitam repetir senhas já usadas.

## Gerar o executável (.exe)

Com o Python instalado no Windows, dê dois cliques em `compilar.bat` ou rode:

```bat
py -m pip install --upgrade pyinstaller customtkinter pywin32

py -m PyInstaller --noconfirm --clean --windowed --onedir ^
  --name "Sistema de impressão" ^
  --icon "icone.ico" ^
  --add-data "icone.ico;." ^
  --collect-all customtkinter ^
  --hidden-import win32print --hidden-import pywintypes ^
  app.pyw
```

O programa fica em `dist\Sistema de impressão\`. Dicas:

- Copie a pasta para um local fixo (ex.: `C:\Posto44\`) antes de usar; uma nova compilação apaga o `dist`.
- Os dados do usuário ficam **ao lado do `.exe`**. Ao atualizar, substitua só o `.exe` e a pasta `_internal`.
- Crie o atalho com **Enviar para → Área de trabalho**; o ícone vem do executável.
- Antivírus podem marcar executáveis do PyInstaller como falso positivo; adicione a pasta às exceções.

## Solução de problemas

| Sintoma | O que verificar |
|---|---|
| "Impressora não encontrada" | O nome em **Config** deve ser idêntico ao do Windows. Use **Atualizar lista**. |
| Nada imprime / erro de impressão | Impressora ligada, com papel e conectada. Veja `erros.log`. |
| Senhas sem confirmação / timeout no GDC | Use **Testar GDC**, confira IP/porta e o cabo de rede. Se o GDC não aguentar o ritmo, aumente a **pausa entre senhas** em Config. |
| Mensagem "O GDC encerrou a conexão" | Aumente a pausa entre senhas. |
| O `.exe` não abre | Veja o `erros.log` ao lado do executável. |
| `No module named win32print` | `pip install pywin32` (ou recompile com os `--hidden-import` acima). |

## Privacidade e segurança

- O `config.json` guarda a **senha do WiFi**; ele está no `.gitignore` e **não deve ser publicado**.
- Antes de tornar o repositório público, troque no código os valores padrão de WiFi (`CONFIG_PADRAO` em `tela_ctk.py`) e de IP do GDC (`gdc.py`) por valores genéricos.

## Criador

- Warlen Kauã

## Cocriador

- Luan Delaflora


