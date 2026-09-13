# Pipeline CI/CD para Containerização e GitOps

Este projeto constrói um pipeline completo de CI/CD em volta de uma aplicação Java de gerenciamento de contas de usuário (a vprofile, uma aplicação de exemplo bastante usada para praticar DevOps, com Spring MVC, RabbitMQ, ElasticSearch, Memcached e MySQL), cobrindo build, análise de qualidade de código, containerização em múltiplas camadas e entrega contínua até um repositório Helm separado, seguindo o modelo GitOps. O foco aqui não é a aplicação Java em si, mas toda a esteira de automação construída ao redor dela para levá-la de código fonte a uma imagem pronta para rodar em produção.

## Como funciona

O pipeline, definido em GitHub Actions, se comporta de forma diferente dependendo do tipo de evento que o dispara. Num pull request, ele entra no modo de validação: compila a aplicação com Maven, roda os testes automatizados, gera um relatório de Checkstyle e envia tudo para uma análise de qualidade no SonarQube, que só libera a integração se passar pelo quality gate configurado. Isso funciona como um portão de qualidade antes que qualquer código entre na branch principal.

Já num push direto na branch principal, o pipeline entra no modo de entrega. Ele configura credenciais da AWS, garante que o repositório no Amazon ECR existe (criando um se necessário) e constrói a imagem Docker usando um Dockerfile multi estágio: o primeiro estágio clona o código fonte da aplicação e compila o artefato com Maven, dentro de uma imagem só com as ferramentas de build, e o segundo estágio parte de uma imagem limpa do Tomcat e copia só o artefato já compilado para dentro dela, sem carregar nenhuma ferramenta de build na imagem final. Essa separação em duas camadas é o que mantém a imagem de produção pequena e sem dependências desnecessárias.

Depois de construída, a imagem é enviada para o Amazon ECR, marcada tanto com o hash do commit quanto com a tag `latest`. A última etapa do pipeline fecha o ciclo de GitOps: ele clona um repositório separado que guarda os manifests do Helm, atualiza o `values.yaml` com o nome da nova imagem e sua tag, e commita essa mudança de volta. Isso desacopla a aplicação da configuração de deploy, um sistema de entrega contínua observando esse segundo repositório consegue disparar a atualização em produção sem precisar de acesso direto ao repositório de código.

Além do Dockerfile multi estágio usado no pipeline, o projeto também tem Dockerfiles separados para as outras camadas da aplicação (banco de dados e proxy web), pensados para rodar o ambiente completo localmente em containers durante o desenvolvimento.

## Estrutura do projeto

```
cloudapp-aplication/
├── src/                          # codigo fonte Java (Spring MVC) da aplicacao
├── Docker-files/
│   ├── app/
│   │   ├── Dockerfile             # build simples, espera o .war ja compilado
│   │   └── multistage/Dockerfile  # build multi estagio usado no pipeline
│   ├── db/                        # container do banco de dados
│   └── web/                       # container do proxy web (nginx)
├── sonar-project.properties       # configuracao da analise SonarQube
├── pom.xml                        # dependencias e build Maven
└── .github/workflows/ci.yml       # pipeline de build, qualidade, imagem e GitOps
```

## Como rodar

O pipeline roda automaticamente a cada pull request ou push na branch principal, mas ele depende de segredos e variáveis configurados no repositório do GitHub: credenciais AWS, o nome do repositório ECR, a URL do SonarQube e o token de acesso ao repositório Helm separado.

Para compilar a aplicação localmente:

```bash
mvn clean install
```

Para construir a imagem de produção da mesma forma que o pipeline faz:

```bash
docker build --file Docker-files/app/multistage/Dockerfile --tag cloudapp:local .
```

## Observações

Esse pipeline reúne várias práticas de entrega contínua num fluxo só: build multi estágio para imagens enxutas, quality gate automatizado antes do merge, publicação versionada de imagens no ECR e atualização automática da configuração de deploy via GitOps, sem precisar de intervenção manual em nenhuma dessas etapas. Um próximo passo natural seria adicionar testes de integração rodando contra os containers de banco de dados e mensageria antes da publicação da imagem, hoje o pipeline valida a aplicação isoladamente, sem o ambiente completo dela em volta.
