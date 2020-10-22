@Library('pmd@family-pmd4') _

import uk.org.floop.jenkins_pmd.Drafter

pipeline {
    agent {
        label 'master'
    }
    stages {
        stage('Clean') {
            steps {
                sh 'rm -rf out'
                sh 'mkdir out'
                sh 'mkdir -p in'
            }
        }
        stage('Fetch') {
            steps {
                script {
                    def response = httpRequest(contentType: 'APPLICATION_ZIP',
                                               httpMode: 'GET',
                                               url: 'https://files.digital.nhs.uk/assets/ods/current/epraccur.zip',
                                               outputFile: 'in/epraccur.zip')
                    unzip zipFile: 'in/epraccur.zip', dir: 'out'
                }
            }
        }
        stage('Convert to RDF') {
            agent {
                docker {
                    image 'gsscogs/csv2rdf'
                    reuseNode true
                    alwaysPull true
                }
            }
            steps {
                script {
                    sh "csv2rdf -m annotated -t out/epraccur.csv -u epraccur.csv-metadata.json -o out/epraccur.ttl"
                }
            }
        }
	      stage('Upload') {
            agent {
                docker {
                    image 'gsscogs/csv2rdf'
                    reuseNode true
                    alwaysPull true
                }
            }
            steps {
                script {
                    def pmd = pmdConfig('pmd')
                    for (myDraft in pmd.drafter
                            .listDraftsets(Drafter.Include.OWNED)
                            .findAll { it['display-name'] == env.JOB_NAME }) {
                        pmd.drafter.deleteDraftset(myDraft.id)
                    }
                    def id = pmd.drafter.createDraftset(env.JOB_NAME).id
                    for (graph in util.jobGraphs(pmd, id)) {
                        pmd.drafter.deleteGraph(id, graph)
                        echo "Removing own graph ${graph}"
                    }
                    def uploads = []
                    writeFile file: "csgraph.sparql", text: """SELECT ?graph { ?graph a <http://www.w3.org/2004/02/skos/core#ConceptScheme> }"""
                    for (def cs : findFiles(glob: 'out/*.ttl')) {
                        sh "sparql --data='${cs.path}' --query=csgraph.sparql --results=JSON > 'graph.json'"
                        uploads.add([
                                "path"  : cs.path,
                                "format": "text/turtle",
                                "graph" : readJSON(text: readFile(file: "graph.json")).results.bindings[0].graph.value
                        ])
                    }
                    for (def upload : uploads) {
                        pmd.drafter.addData(id, "${WORKSPACE}/${upload.path}", upload.format, "UTF-8", upload.graph)
                        writeFile(file: "${upload.path}-prov.ttl", text: util.jobPROV(upload.graph))
                        pmd.drafter.addData(id, "${WORKSPACE}/${upload.path}-prov.ttl", "text/turtle", "UTF-8", upload.graph)
                    }
                    pmd.drafter.publishDraftset(id)
                }
            }
        }
    }
}
