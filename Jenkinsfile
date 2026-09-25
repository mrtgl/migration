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

def enc(String s) {
    return URLEncoder.encode(s, 'UTF-8').replace('+', '%20')
}

def papi(String method, String url, String body, boolean allowFail = false) {
    def args = [url: url, httpMode: method, acceptType: 'APPLICATION_JSON',
                validResponseCodes: '100:599', ignoreSslErrors: true, quiet: true]
    if (body != null) {
        args.contentType = 'APPLICATION_JSON'
        args.requestBody = body
    }
    def resp = httpRequest(args)
    echo "${method} ${url} -> ${resp.status}"
    if (!allowFail && (resp.status < 200 || resp.status > 299)) {
        error "İstek başarısız (${resp.status}): ${resp.content}"
    }
    return resp
}

// İsimle arar; bulamazsa birkaç kez tekrar dener
def lookupUuid(String base, String kind, String name, int tries) {
    def last = ''
    for (int i = 0; i < tries; i++) {
        def resp = papi('GET', "${base}/api-management/1.0/${kind}?name=${enc(name)}", null, true)
        last = resp.content
        def j = parse(resp.content)
        def list = (j instanceof List) ? j : (j?.results ?: [])
        for (item in list) {
            if (item.name == name) return item.uuid
        }
        if (i < tries - 1) sleep 3
    }
    echo "Bulunamadı (${name}). Son GET cevabı: ${last?.take(1000)}"
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
                        def isProductCreate = r.method == 'POST' && r.path.endsWith('/products')
                        def kind = r.path.endsWith('/apis') ? 'apis' :
                                   r.path.endsWith('/rate-quotas') ? 'rate-quotas' : null
                        def isNamedCreate = r.method == 'POST' && kind

                        // Zaten varsa atla
                        if (isProductCreate) {
                            def chk = papi('GET', "${base}/api-management/1.0/products/${r.body.uuid}", null, true)
                            if (chk.status == 200) {
                                echo "Product zaten var, atlanıyor: ${r.body.name}"
                                continue
                            }
                        }
                        if (isNamedCreate) {
                            def existing = lookupUuid(base, kind, r.body.name, 1)
                            if (existing) {
                                ids[(kind == 'apis' ? 'api:' : 'rateQuota:') + r.body.name] = existing
                                echo "Zaten var, atlanıyor: ${r.body.name} -> ${existing}"
                                continue
                            }
                        }

                        def url = base + resolve(r.path, ids)
                        if (r.params) {
                            def q = []
                            for (k in r.params.keySet()) {
                                q << enc(k) + '=' + enc(r.params[k].toString())
                            }
                            url += '?' + q.join('&')
                        }

                        def body = r.containsKey('body') ? resolve(JsonOutput.toJson(r.body), ids) : null
                        def isPatch = r.method == 'PATCH'
                        def resp = papi(r.method, url, body, isPatch)
                        if (isPatch && (resp.status < 200 || resp.status > 299)) {
                            echo "UYARI: PATCH ${resp.status} döndü (API'ler zaten product'ta olabilir): ${resp.content}"
                        }

                        if (isNamedCreate) {
                            def name = r.body.name
                            def created = parse(resp.content)
                            def uuid = (created instanceof Map ? created.uuid : null) ?: lookupUuid(base, kind, name, 5)
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
