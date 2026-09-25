import groovy.json.JsonOutput
import groovy.json.JsonSlurperClassic

def parse(String text) {
    return text?.trim() ? new JsonSlurperClassic().parseText(text) : null
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

// Yoksa oluşturur, varsa mevcut UUID'yi döner
def createNamed(String base, String kind, String path, Map body) {
    def existing = lookupUuid(base, kind, body.name, 1)
    if (existing) {
        echo "Zaten var, atlanıyor: ${body.name} -> ${existing}"
        return existing
    }
    def resp = papi('POST', base + path, JsonOutput.toJson(body))
    def created = parse(resp.content)
    def uuid = (created instanceof Map ? created.uuid : null) ?: lookupUuid(base, kind, body.name, 5)
    if (!uuid) error "UUID bulunamadı: ${body.name}"
    echo "${body.name} -> ${uuid}"
    return uuid
}

pipeline {
    agent any

    parameters {
        string(name: 'PAPI_BASE_URL', defaultValue: 'http://10.10.4.11:8080/papi',
               description: 'PAPI adresi')
        string(name: 'PRODUCT_DIR', defaultValue: 'l7out/helloworld_0452',
               description: 'product.json, apis/ ve rate-quotas/ içeren klasör')
    }

    stages {
        stage('PAPI import') {
            steps {
                script {
                    def base = params.PAPI_BASE_URL.replaceAll('/+$', '')
                    def dir = params.PRODUCT_DIR.replaceAll('/+$', '')
                    def mgmt = '/api-management/1.0'
                    def result = [apis: [:], rateQuotas: [:]]

                    // 1) Product
                    def product = parse(readFile("${dir}/product.json"))
                    def chk = papi('GET', "${base}${mgmt}/products/${product.uuid}", null, true)
                    if (chk.status == 200) {
                        echo "Product zaten var, atlanıyor: ${product.name}"
                    } else {
                        papi('POST', "${base}${mgmt}/products", JsonOutput.toJson(product))
                    }
                    result.product = product.uuid

                    // 2) API'ler: oluştur -> policy-entities -> publish
                    def apiUuids = []
                    def apiFiles = findFiles(glob: "${dir}/apis/*.json")
                    for (f in apiFiles) {
                        if (f.name.endsWith('.policy-entities.json')) continue

                        def api = parse(readFile(f.path))
                        def uuid = createNamed(base, 'apis', "${mgmt}/apis?addWildcard=false", api)

                        def policyPath = f.path.replaceAll(/\.json$/, '.policy-entities.json')
                        if (fileExists(policyPath)) {
                            papi('PUT', "${base}${mgmt}/apis/${uuid}/policy-entities", readFile(policyPath))
                        } else {
                            echo "UYARI: policy dosyası yok: ${policyPath}"
                        }

                        papi('PUT', "${base}${mgmt}/apis/${uuid}/publish", null)
                        apiUuids << uuid
                        result.apis[api.name] = uuid
                    }

                    // 3) API'leri product'a bağla
                    if (apiUuids) {
                        def links = []
                        for (u in apiUuids) links << [apiUuid: u]
                        def resp = papi('PATCH', "${base}${mgmt}/products/${product.uuid}/apis?action=ADD",
                                        JsonOutput.toJson(links), true)
                        if (resp.status < 200 || resp.status > 299) {
                            echo "UYARI: PATCH ${resp.status} döndü (API'ler zaten product'ta olabilir): ${resp.content}"
                        }
                    }

                    // 4) Rate-quota'lar
                    def rqFiles = findFiles(glob: "${dir}/rate-quotas/*.json")
                    for (f in rqFiles) {
                        def rq = parse(readFile(f.path))
                        result.rateQuotas[rq.name] = createNamed(base, 'rate-quotas', "${mgmt}/rate-quotas", rq)
                    }

                    writeFile file: 'import-result.json', text: JsonOutput.prettyPrint(JsonOutput.toJson(result))
                }
            }
        }
    }

    post {
        always { archiveArtifacts artifacts: 'import-result.json', allowEmptyArchive: true }
    }
}
