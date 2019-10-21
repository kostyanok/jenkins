throttle(['one-per-node-throttle']) {
    node('linux') {
        checkout scm
        pipeline = load 'JenkinsfileScript.groovy'
        pipeline.call()
    }
}
