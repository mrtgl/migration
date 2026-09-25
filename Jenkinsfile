import groovy.json.JsonOutput
import groovy.json.JsonSlurperClassic

def parse(String text) {
    return text?.trim() ? new JsonSlurperClassic().parseText(text) : null
}

// ${api:...} / ${rateQuota:...} yer tutucularını oluşan UUID'lerle değiştirir
def resolve(String text, Map ids) {
    for (k in ids.keySet()) {
        text = text.replace('${' + k + '}', ids[k])
    }
    if (text.contains('${api:') || text.contains('${rateQuota:')) {
        error "Çözülemeyen yer tutucu: ${text.take(300)}"
    }
    return text
}

def papi(String method, String url, String body) {
    def args = [url: url, httpMode: method, acceptType: 'APPLICATION_JSON',
                validResponseCodes: '100:599', ignoreSslErrors: true, quiet: true]
    if (body != null) {
        args.contentType = 'APPLICATION_JSON'
        args.requestBody = body
    }
    def resp = httpRequest(args)
    echo "${method} ${url} -> ${resp.status}"
    if (resp.status < 200 || resp.status > 299) {
        error "İstek başarısız (${resp.status}): ${resp.content}"
    }
    return resp
}

// POST cevabında uuid yoksa isimle arayıp bulur
def lookupUuid(String base, String kind, String name) {
    def resp = papi('GET', "${base}/api-management/1.0/${kind}?name=${URLEncoder.encode(name, 'UTF-8')}", null)
    def j = parse(resp.content)
    def list = (j instanceof List) ? j : (j?.results ?: [])
    for (item in list) {
        if (item.name == name) return item.uuid
    }
    return null
}

pipeline {
    agent any

    parameters {
        string(name: 'PAPI_BASE_URL', defaultValue: 'http://10.10.4.11:8080/papi',
               description: 'PAPI adresi')
        string(name: 'PLAN_FILE', defaultValue: 'l7out/requests-plan-0452.json',
               description: 'Workspace içindeki requests-plan dosyası')
    }

    stages {
        stage('PAPI import') {
            steps {
                script {
                    def base = params.PAPI_BASE_URL.replaceAll('/+$', '')
                    def plan = parse(readFile(params.PLAN_FILE))
                    def ids = [:]

                    for (r in plan.requests) {
                        def url = base + resolve(r.path, ids)
                        if (r.params) {
                            def q = []
                            for (k in r.params.keySet()) {
                                q << URLEncoder.encode(k, 'UTF-8') + '=' + URLEncoder.encode(r.params[k].toString(), 'UTF-8')
                            }
                            url += '?' + q.join('&')
                        }

                        def body = r.containsKey('body') ? resolve(JsonOutput.toJson(r.body), ids) : null
                        def resp = papi(r.method, url, body)

                        def kind = r.path.endsWith('/apis') ? 'apis' :
                                   r.path.endsWith('/rate-quotas') ? 'rate-quotas' : null
                        if (r.method == 'POST' && kind) {
                            def name = r.body.name
                            def created = parse(resp.content)
                            def uuid = (created instanceof Map ? created.uuid : null) ?: lookupUuid(base, kind, name)
                            if (!uuid) error "UUID bulunamadı: ${name}"
                            ids[(kind == 'apis' ? 'api:' : 'rateQuota:') + name] = uuid
                            echo "${name} -> ${uuid}"
                        }
                    }

                    writeFile file: 'import-result.json', text: JsonOutput.prettyPrint(JsonOutput.toJson(ids))
                }
            }
        }
    }

    post {
        always { archiveArtifacts artifacts: 'import-result.json', allowEmptyArchive: true }
    }
}
