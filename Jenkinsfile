@Library('jenkins-test-library') _

def configMap = [
    project: "roboshop-2",
    component: "catalogue-2"
]

echo "Triggering the library pipeline"

if ( env.BRANCH_NAME.equalsIgnoreCase('main') ){
     echo "checking later"
}
else{
    testPipeline(configMap)
}
