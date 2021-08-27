# amq-streams-foundations-labs
lab assets for the 4 modules of AMQ streams foundations course

Lab Instructions Location:

`https://start.learning.redhat.com/totara/1626`

Install Red Hat AMQ Streams:

Download and extract the install_and_examples.zip file from the (download site)[https://www.opentlc.com/download/amq_packages/].

Create a project if it does not exist

`oc new-project kafkaproject`

Navigate to extracted directory and execute sed to put in correct namespace

`sed -i 's/namespace: .*/namespace: kafkaproject/' install/cluster-operator/*RoleBinding*.yaml`

Apply the manifests in the install/cluster-operator directory from within the extracted directory

`oc apply -f install/cluster-operator -n kafkaproject`

Apply the kafka templates manfiests

`oc apply -f examples/templates/cluster-operator/ -n kafkaproject`
