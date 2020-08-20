# Working with a Pipeline

## Components 
Now that we know how to work with tasks and their params it is a good time to move on and start working with pipelines  
Pipelines are consistent of several parts :

  1. pipeline resources
  2. pipeline pvc
  3. workspace
  4. pipeline (metadata definition)
  5. pipeline Run 

  In this part we will go over each of the components and understand how they all work together to create a healthy pipeline.

## Getting dirty

### Planing the Pipeline

I know it may sound Obvious but a good pipeline needs a good planing before we even write the first row  
We need to know what are our resources , we need to know what are our tasks and we need to know if we have an option to run  
several tasks in parallel or do we need to run them sequentially 

### Planning

In this part we are going to build a pipeline that will work with a git repository as it's input resource. In the pipeline we will  
run a task that will build a simple go application , save it to a pvc and will create the application in OpenShift.(sound simple right?)  

### Configuring the Resource

As mentioned we will start by configuring the pipeline resource git which will be used for our Source Resource.  
In order to configure it we will apply the following YAML:  

    #cat > pipelineResource-git.yaml << EOF
    apiVersion: tekton.dev/v1alpha1
    kind: PipelineResource
    metadata:
      name: monkey-app-git
    spec:
      type: git
      params:
        - name: revision
          value: master
        - name: url
          value: https://github.com/ooichman/pipeline-tutorial.git
    EOF

Now we can create another resource which will be our output resource , In our case we will create an output resource which will be our application Image.

First export your namespace :

    #export NAMESPACE=`oc project -q`

Now we will create the pipeline resource YAML file.

    # cat > pipelineResource-image.yaml << EOF
    apiVersion: tekton.dev/v1alpha1
    kind: PipelineResource
    metadata:
      name: monkey-app
    spec:
      type: image
      params:
      - name: url
        value: image-registry.openshift-image-registry.svc:5000/${NAMESPACE}/monkey-app:latest
    EOF

Another resource that we are going to use in our case is a PVC to store our image after it is build.  
For that we are going to create our PVC as follow :

    #cat > pipeline-pvc.yaml << EOF
    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: container-build
      namespace: $NAMESPACE
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 5Gi
      volumeMode: Filesystem
    EOF

Once we have created all our Resources files we can create then on OpenShift 

    #oc create -f pipelineResource-git.yaml -f pipelineResouce-image.yaml -f pipeline-pvc.yaml

Now we can start with the FUN part which is the build process

### Pipeline Tasks

Our pipeline needs tasks so it will know what to do.  
In our pipeline we need to stop and think about what it needs to do , taking into pieces and build a task from each piece.  

So to make our job easier we basically need to :

  1. build the application from the git Repository
  2. push it to our registry
  3. deploy the application using our YAML files

  so for the first task we can use an image named buildah which we can obtain from quay.io to build the image :

our task should look like :

    #echo 'apiVersion: tekton.dev/v1beta1
    kind: Task
    metadata:
      name: monkey-build-task
    spec:
    ##################### New content ###################
      resources:
        inputs:
          - {type: git, name: source}
        outputs:
          - {type: image , name: image}
    ##################### New content ###################
      params:
        - name: image-name
          description: The Name of the Image we want to use
          type: string
          default: "monkey-app"
      steps:
    ##################### New content ###################
        - name: build
          image: quay.io/buildah/stable:v1.11.0
          workingDir: /workspace/source/
          volumeMounts:
          - name: varlibcontainers
            	mountPath: /var/lib/containers
          command: ["/bin/bash" ,"-c"]
          args:
            - |-
              buildah bud -f Dockerfile -t $(resources.outputs.image) .
      volumes:
      - name: varlibcontainers
        persistentVolumeClaim:
          claimName: container-build
    ##################### New content ###################' > task-build-image.yaml

As you can see we added a few more components we haven't used so far.  

in our spec definition we've added the "resources" schema.  
That is enabling us to use our pipeline resource (which we defined earlier) in our task which in this case we are using our git repository and the image as our output.  
  
Next we are using a param option to define our application name.  
For the last part use can see that we are defining our PVC as our mount directory and then we are running our buildah application.

Now let's create the task :

    #oc create -f task-build-image.yaml

### The Pipeline (sequential) 

now that we have 3 tasks in place we can start and build the pipeline :

    #cat > pipeline-build-monkey.yaml << EOF
    apiVersion: tekton.dev/v1beta1
    kind: Pipeline
    metadata:
    ##################### the pipeline information ###################
      name: build-monkey
    spec:
      resources:
    ##################### the pipeline resource definition ###################
      - name: source
        type: git
      - name: image
        type: image
      tasks:
    ##################### the tasks reference ###################
      - name: hello-world
        taskRef:
          name: echo-hello-world
      - name: build-image
        taskRef:
          name: monkey-build-task
        runAfter: 
          - hello-world
        resources:
          inputs:
          - name: source
            resource: source
          outputs:
          - name: image
            resource: image
      - name: hello-person
        taskRef:
          name: echo-hello-person
        runAfter:
          - monkey-build-task
    EOF

I know for the first time looking at the task it can look very complex but believe me , it isn't.  

when we look at the YAML file we can see there is actually just 3 parts

  1. the Pipeline information
  2. the resources
  3. the tasks (in a sequential order)

#### Pipeline information

As you can see the pipeline information (at this point) is very small and simple. All we need to give here is the name of the task we want to run.  

#### the Pipeline Resource

At the beginning of this part we are defining the resources we want to use. In this example we are using a GIT resource named "source" and a IMAGE resource named image (the names must match between the pipeline and the task)

#### the Tasks

In the last section we are defining the task (in a sequential order ) and if the task requires a resource then we are defining the resource to that particular task.  
  
All that is left right now is to create a pipeline run with a reference to the pipeline resources we define at the beginning of the chapter :

    # cat > pipeline-run-build-monkey.yaml << EOF
    apiVersion: tekton.dev/v1alpha1
    kind: PipelineRun
    metadata:
      name: build-pipeline
    spec:
      # serviceAccountName: pipeline
      pipelineRef:
        name: pipeline-run-build-monkey
      inputs:
        resources:
          - name: source
            resourceRef:
              name: monkey-app-git
      outputs:
        resources:
          - name: image
            resourceRef:
              name: monkey-app
    EOF

And create the object with oc

    #oc create -f pipeline-run-build-monkey.yaml
    (or)
    #tkn pipeline start build-pipeline

Follow the logs and see the magic happens...  

### WorkSpace

A Pipeline can use Workspaces to show how storage will be shared through its Tasks. For example, Task A might   clone a source repository onto a Workspace and Task B might compile the code that it finds in that Workspace.   It’s the Pipeline's job to ensure that the Workspace these two Tasks use is the same, and more importantly, that  the order in which they access the Workspace is correct.  
  
PipelineRuns perform mostly the same duties as TaskRuns - they provide the specific Volume information to use   for the Workspaces used by each Pipeline.PipelineRuns have the added responsibility of ensuring that whatever   Volume type they provide can be safely and correctly shared across multiple Tasks.  
  

In order to configure the Workspace we will add the definition :


    #cat > pipeline-build-monkey-ws.yaml << EOF
    apiVersion: tekton.dev/v1beta1
    kind: Pipeline
    metadata:
      name: build-monkey
    spec:
      resources:
      - name: source
        type: git
      - name: image
        type: image
      tasks:
      - name: hello-world
        taskRef:
          name: echo-hello-world
      - name: build-image
        taskRef:
          name: monkey-build-task
        runAfter: 
          - hello-world
################### Workspace Definition ##########
        workspaces:
        - name: src
          workspace: pipeline-ws1
################### Workspace Definition Ends #####
        resources:
          inputs:
          - name: source
            resource: source
          outputs:
          - name: image
            resource: image
      - name: hello-person
        taskRef:
          name: echo-hello-person
        runAfter: 
          - monkey-build-task
    EOF

And we will add a reference for a pipeline run

    # cat > pipeline-run-build-monkey.yaml << EOF
    apiVersion: tekton.dev/v1alpha1
    kind: PipelineRun
    metadata:
      name: build-pipeline
    spec:
      workspaces:
      - name: pipeline-ws1
        persistentVolumeClaim:
          claimName: container-build
      # serviceAccountName: pipeline
      pipelineRef:
        name: pipeline-run-build-monkey
      inputs:
        resources:
          - name: source
            resourceRef:
              name: monkey-app-git
      outputs:
        resources:
          - name: image
            resourceRef:
              name: monkey-app
    EOF


#### Open task

Build your own task that will push the image to the registry  
(Hint: you can use the "quay.io/buildah/stable:v1.11.0" Image)

### The Pipeline (parallel)

The Only Difference between sequential and parallel is the "runAfter" section.
recreate the task without the runAfter, create a pipeline run and let me know what you notice

We will wait for the rest of the Class to complete the exercise and move on to [Exercise 3](../Exercise-3/Exercise-3.md)