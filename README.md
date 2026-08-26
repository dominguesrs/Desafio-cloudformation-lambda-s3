# Desafio: Automação de Infraestrutura AWS com CloudFormation, Lambda e S3

Repositório de estudos e prática consolidando os conceitos de **Infrastructure as Code (IaC)** na AWS, usando o **CloudFormation** para provisionar recursos de forma automatizada, indo de stacks simples até cenários com rede, segurança, automação event-driven (Lambda + S3) e versionamento.

## 🎯 Objetivos de aprendizagem

- Aplicar conceitos de IaC em um ambiente prático na AWS
- Documentar processos técnicos de forma clara e estruturada
- Praticar automação Lambda + S3 (execução orientada a eventos)
- Usar o GitHub como ferramenta de compartilhamento de documentação técnica

## 📚 Base de referência

A documentação oficial usada como ponto de partida foi o guia da AWS sobre [automatizar S3 Object Lambda com CloudFormation](https://docs.aws.amazon.com/pt_br/AmazonS3/latest/userguide/olap-using-cfn-template.html).

> ⚠️ **Nota importante:** desde 07/11/2025 o recurso S3 Object Lambda (Access Points) passou a ser restrito apenas a clientes que já utilizavam o serviço, além de parceiros selecionados da AWS Partner Network. Contas novas não conseguem provisionar esse recurso específico. Por isso, os templates deste repositório aplicam o mesmo princípio de automação Lambda + S3 (execução orientada a eventos, IAM, notificações) usando recursos que continuam disponíveis para qualquer conta: notificações de bucket S3 (`NotificationConfiguration`) disparando funções Lambda.

## 🏗️ Estrutura das stacks

| # | Stack | Recursos | Conceitos praticados |
|---|-------|----------|----------------------|
| 1 | [`01-s3-bucket-simples.yaml`](templates/01-s3-bucket-simples.yaml) | 1 bucket S3 | Sintaxe básica do CFN, Outputs, criptografia padrão, bloqueio de acesso público |
| 2 | [`02-lambda-s3-trigger.yaml`](templates/02-lambda-s3-trigger.yaml) | Bucket S3 + Lambda + IAM Role | Automação event-driven, permissões entre serviços, `DependsOn` para resolver dependência circular S3↔Lambda |
| 3 | [`03-vpc-ec2-seguranca.yaml`](templates/03-vpc-ec2-seguranca.yaml) | VPC, subnet pública, IGW, tabela de rotas, Security Group, EC2 | Rede desde o zero, boas práticas de segurança (SSH restrito ao IP do autor), resolução dinâmica de AMI via SSM Parameter (`AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>`) para evitar IDs de AMI fixos e desatualizados |
| 4 | [`04-s3-versionamento-completo.yaml`](templates/04-s3-versionamento-completo.yaml) | Bucket principal + bucket de logs + Lambda de auditoria | Versionamento, regras de ciclo de vida (lifecycle), logging de acesso, automação de auditoria por versão |

## 🚀 Como deployar cada stack

Substitua `<nome-unico>` por um valor único (nomes de bucket S3 são globais).

```bash
# Stack 1
aws cloudformation deploy \
  --template-file templates/01-s3-bucket-simples.yaml \
  --stack-name desafio-stack1-s3 \
  --parameter-overrides NomeDoBucket=<nome-unico>-stack1

# Stack 2
aws cloudformation deploy \
  --template-file templates/02-lambda-s3-trigger.yaml \
  --stack-name desafio-stack2-lambda-s3 \
  --parameter-overrides NomeDoBucket=<nome-unico>-stack2 \
  --capabilities CAPABILITY_IAM

# Stack 3 (exige um Key Pair já existente na região e seu IP público)
aws cloudformation deploy \
  --template-file templates/03-vpc-ec2-seguranca.yaml \
  --stack-name desafio-stack3-vpc-ec2 \
  --parameter-overrides ChavePar=<seu-keypair> MeuIpParaSSH=<seu-ip>/32

# Stack 4
aws cloudformation deploy \
  --template-file templates/04-s3-versionamento-completo.yaml \
  --stack-name desafio-stack4-versionamento \
  --parameter-overrides NomeBucketPrincipal=<nome-unico>-principal NomeBucketLogs=<nome-unico>-logs \
  --capabilities CAPABILITY_IAM
```

Para remover tudo ao final (evitar custos):

```bash
aws cloudformation delete-stack --stack-name <nome-da-stack>
```

## 🖼️ Evidências

Todas as capturas de tela ficam em [`/images`](images), organizadas por prefixo `stackN-` para relacionar cada imagem à stack correspondente.

### Stack 1 — S3 bucket simples
| Arquivo | O que mostra |
|---|---|
| `stack1-01-status-create-complete.png` | Stack com status `CREATE_COMPLETE` |
| `stack1-02-resources.png` | Aba Resources com o bucket criado |
| `stack1-03-outputs.png` | Outputs (nome e ARN do bucket) |
| `stack1-04-bucket-properties.png` | Versionamento e criptografia habilitados |
| `stack1-05-block-public-access.png` | Bloqueio de acesso público ativo |
| `stack1-06-versionamento-teste.png` | Teste real de upload gerando múltiplas versões |

### Stack 2 — Lambda + S3 (automação event-driven)
| Arquivo | O que mostra |
|---|---|
| `stack2-01-status-create-complete.png` | Stack com status `CREATE_COMPLETE` |
| `stack2-02-resources.png` | Aba Resources (bucket, role, função, permissão) |
| `stack2-03-event-notification.png` | Notificação de evento configurada no bucket |
| `stack2-04-lambda-code.png` | Código da função Lambda |
| `stack2-05-cloudwatch-logs.png` | Log de execução após upload de teste |

### Stack 3 — VPC, rede e segurança
| Arquivo | O que mostra |
|---|---|
| `stack3-01-status-outputs.png` | Stack `CREATE_COMPLETE` e Outputs (IP público, ID da VPC) |
| `stack3-02-resources.png` | Aba Resources com os 8 recursos de rede |
| `stack3-03-vpc-subnet.png` | Detalhes da subnet pública (`Auto-assign public IPv4: Yes`) |
| `stack3-04-resource-map.png` | Mapa de recursos da VPC (subnet, route tables e IGW conectados) |
| `stack3-05-security-group.png` | Regras de entrada do Security Group (80 aberto, 22 restrito ao IP do autor) |
| `stack3-06-teste-http.png` | Página do Apache acessível via IP público da EC2 |

### Stack 4 — Versionamento completo + auditoria
| Arquivo | O que mostra |
|---|---|
| `stack4-01-status-outputs.png` | Stack `CREATE_COMPLETE` e Outputs (buckets criados) |
| `stack4-02-resources.png` | Aba Resources com os 6 recursos da stack |
| `stack4-03-versoes.png` | Múltiplas versões do mesmo objeto no bucket principal |
| `stack4-04-logs-auditoria.png` | Logs da Lambda de auditoria registrando cada versão |
| `stack4-05-lifecycle.png` | Regra de lifecycle (transição para IA e expiração) |
| `stack4-06-logging-config.png` | Configuração de logging de acesso apontando para o bucket de logs |

## 📝 Insights e aprendizados

Anotações sobre erros encontrados, decisões de design e o que cada stack ensinou estão em [`docs/insights.md`](docs/insights.md) — preenchido durante a prática.

## 🧰 Pré-requisitos

- Conta AWS com permissões para CloudFormation, S3, Lambda, IAM, EC2 e VPC
- AWS CLI configurado (`aws configure`)
- Um Key Pair EC2 já criado na região usada (para a Stack 3)

## 📄 Licença

Uso livre para fins de estudo.
