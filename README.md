# Ferramentas de Segurança na Pipeline CI/CD

## Introdução
Este repositório demonstra como integrar ferramentas de análise estática (SAST) e análise dinâmica (DAST) em uma pipeline CI/CD usando GitHub Actions.

## Objetivo
Os principais objetivo de implementações desse tipo são:
- Identificar vulnerabilidades na codebase antes que possam ser exploradas em produção;
- Melhorar a postura geral de segurança da aplicação;
- Automatizar certas verificações de segurança para agilizar o processo de desenvolvimento e dar escala à times de segurança.

## Implementação do Workflow

O workflow do GitHub Actions é estruturada da seguinte forma:
1. **Eventos**
   - O workflow é disparado nos eventos de `push` e `pull request` no repositório da aplicação. Também é possível executar sob demanda quando o dono do repositório solicitar.

2. **Jobs**
   - **Build:** O primeiro job faz o build da aplicação.
   - **SAST:** Após a conclusão do job de build, os testes de SAST são executados. Este job analisa o código estaticamente.
   - **DAST:** Uma vez que o job SAST é concluído com sucesso, a verificação DAST é executada para identificar vulnerabilidades em tempo de execução. Por se tratar de um teste dinâmico, scans DAST geralmente levam um pouco mais de tempo para executar que scans SAST.

Vale ressaltar que não foram configurados rulesets customizados. Os scans estão rodando com suas configurações padrão e, portanato, não estão 100% otimizados.

## Ferramentas Usadas
1. **SAST :: Semgrep**
   - **Objetivo:** Ferramentas SAST analisam o código em busca de vulnerabilidades sem executar a aplicação. Elas ajudam a identificar problemas diversos (ex.: SQLi, XSS, Clickjacking, etc) no código.
   - **Implementação:** A ferramenta SAST (Semgrep) está integrada no workflow do GitHub Actions. O workflow ignora qualquer PR aberto pelo DependaBot.
```yaml
sast:
    name: sast/semgrep
    runs-on: ubuntu-latest
    container:
      image: returntocorp/semgrep
    if: (github.actor != 'dependabot[bot]')
    permissions:
      security-events: write
      actions: read
      contents: read
      
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Perform Semgrep Analysis
        run: semgrep scan -q --sarif --config auto . > semgrep-results.sarif

      - name: Save SARIF results as artifact
        uses: actions/upload-artifact@v3
        with:
          name: semgrep-scan-results
          path: semgrep-results.sarif

      - name: Upload SARIF result to the GitHub Security Dashboard
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: semgrep-results.sarif
        if: always()
```

2. **DAST :: ZAP**
   - **Objetivo:** Ferramentas DAST testam a aplicação em tempo de execução em busca de vulnerabilidades, simulando ataques. Elas identificam falhas como problemas de authZ e authN, gerenciamento inadequado de sessões e configurações inseguras no ambiente.
   - **Implementação:** A ferramenta DAST (ZAP) também está integrada no workflow do GitHub Actions. Ela analisa a aplicação em busca de vulnerabilidades, e os resultados são escritos em um arquivo JSON.
```yaml
  dast:
    name: dast/zap
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Download OWASP ZAP Proxy
        run: |
          wget https://github.com/zaproxy/zaproxy/releases/download/v2.15.0/ZAP_2.15.0_Crossplatform.zip
          unzip ZAP_2.15.0_Crossplatform.zip
          cd ZAP_2.15.0

      - name: Build Image
        run: docker compose build
      - name: Launch app
        run: docker compose up --detach
      - name: Check containers
        run: docker ps
      - name: Wait for app to be ready
        run: |
          until curl -s http://localhost:5002; do
            echo "Waiting for the app to start..."
            sleep 5
          done

      - name: Run OWASP ZAP Scan
        env:
          ZAP_URL: 'http://localhost:5002'
        run: |
          cd ZAP_2.15.0
          ./zap.sh -cmd -quickurl "$ZAP_URL" -quickout /home/runner/work/vampi-sectests/vampi-sectests/ZAP_2.15.0/zap-proxy-report.json

      - name: Upload ZAP Report
        uses: actions/upload-artifact@v4
        with:
          name: zap-proxy-report
          path: /home/runner/work/vampi-sectests/vampi-sectests/ZAP_2.15.0/zap-proxy-report.json
```

## Insumos
Os insumos dos scans são:
- Para o SAST, devido ao Semgrep servir os resultados de scan em formato SARIF, o GitHub consegue importar esses dados e organiza-los na aba de [Security](https://github.com/izabela-am/vampi-sectests/security/code-scanning) do repositório. Um exemplo:

![semgrep scan results](https://i.imgur.com/fgODlfO.png)
 
- No caso do DAST, como o ZAP não tem suporte para resultados em SARIF, o GitHub não consegue fazer esse tratamento. É possível fazer o download do JSON com as informações através dos logs das Actions bem-sucedidas. Para facilitar, esse é um output de um dos scans feitos na aplicação:

```json
{
	"@programName": "ZAP",
	"@version": "2.15.0",
	"@generated": "Thu, 3 Oct 2024 18:25:37",
	"site":[ 
		{
			"@name": "http://localhost:5002",
			"@host": "localhost",
			"@port": "5002",
			"@ssl": "false",
			"alerts": [ 
				{
					"pluginid": "10036",
					"alertRef": "10036",
					"alert": "Server Leaks Version Information via \"Server\" HTTP Response Header Field",
					"name": "Server Leaks Version Information via \"Server\" HTTP Response Header Field",
					"riskcode": "1",
					"confidence": "3",
					"riskdesc": "Low (High)",
					"desc": "<p>The web/application server is leaking version information via the \"Server\" HTTP response header. Access to such information may facilitate attackers identifying other vulnerabilities your web/application server is subject to.</p>",
					"instances":[ 
						{
							"uri": "http://localhost:5002",
							"method": "GET",
							"param": "",
							"attack": "",
							"evidence": "Werkzeug/2.2.3 Python/3.11.10",
							"otherinfo": ""
						},
						{
							"uri": "http://localhost:5002/robots.txt",
							"method": "GET",
							"param": "",
							"attack": "",
							"evidence": "Werkzeug/2.2.3 Python/3.11.10",
							"otherinfo": ""
						},
						{
							"uri": "http://localhost:5002/sitemap.xml",
							"method": "GET",
							"param": "",
							"attack": "",
							"evidence": "Werkzeug/2.2.3 Python/3.11.10",
							"otherinfo": ""
						}
					],
					"count": "3",
					"solution": "<p>Ensure that your web server, application server, load balancer, etc. is configured to suppress the \"Server\" header or provide generic details.</p>",
					"otherinfo": "",
					"reference": "<p>https://httpd.apache.org/docs/current/mod/core.html#servertokens</p><p>https://learn.microsoft.com/en-us/previous-versions/msp-n-p/ff648552(v=pandp.10)</p><p>https://www.troyhunt.com/shhh-dont-let-your-response-headers/</p>",
					"cweid": "200",
					"wascid": "13",
					"sourceid": "6"
				},
				{
					"pluginid": "10021",
					"alertRef": "10021",
					"alert": "X-Content-Type-Options Header Missing",
					"name": "X-Content-Type-Options Header Missing",
					"riskcode": "1",
					"confidence": "2",
					"riskdesc": "Low (Medium)",
					"desc": "<p>The Anti-MIME-Sniffing header X-Content-Type-Options was not set to 'nosniff'. This allows older versions of Internet Explorer and Chrome to perform MIME-sniffing on the response body, potentially causing the response body to be interpreted and displayed as a content type other than the declared content type. Current (early 2014) and legacy versions of Firefox will use the declared content type (if one is set), rather than performing MIME-sniffing.</p>",
					"instances":[ 
						{
							"uri": "http://localhost:5002",
							"method": "GET",
							"param": "x-content-type-options",
							"attack": "",
							"evidence": "",
							"otherinfo": "This issue still applies to error type pages (401, 403, 500, etc.) as those pages are often still affected by injection issues, in which case there is still concern for browsers sniffing pages away from their actual content type.\nAt \"High\" threshold this scan rule will not alert on client or server error responses."
						}
					],
					"count": "1",
					"solution": "<p>Ensure that the application/web server sets the Content-Type header appropriately, and that it sets the X-Content-Type-Options header to 'nosniff' for all web pages.</p><p>If possible, ensure that the end user uses a standards-compliant and modern web browser that does not perform MIME-sniffing at all, or that can be directed by the web application/web server to not perform MIME-sniffing.</p>",
					"otherinfo": "<p>This issue still applies to error type pages (401, 403, 500, etc.) as those pages are often still affected by injection issues, in which case there is still concern for browsers sniffing pages away from their actual content type.</p><p>At \"High\" threshold this scan rule will not alert on client or server error responses.</p>",
					"reference": "<p>https://learn.microsoft.com/en-us/previous-versions/windows/internet-explorer/ie-developer/compatibility/gg622941(v=vs.85)</p><p>https://owasp.org/www-community/Security_Headers</p>",
					"cweid": "693",
					"wascid": "15",
					"sourceid": "6"
				}
			]
		}
	]
}

```

