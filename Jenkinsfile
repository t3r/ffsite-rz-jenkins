#!groovy

def GLUON_RELEASE_TAG = "v2016.2.5"
def GLUON_URL = "https://github.com/freifunk-gluon/gluon.git"
def GLUON_BRANCH = "experimental"

def targets = [
  'ar71xx-generic',
  'ar71xx-nand',
  'brcm2708-bcm2708',
  'brcm2708-bcm2709',
  'mpc85xx-generic',
  'x86-generic',
  'x86-kvm_guest',
  'x86-64',
  'x86-xen_domu'
]

node {
    def workspace = pwd()
    stage ("Prepare") {

        /* checkout site config */
        dir('site') {
          checkout scm
        }

        /* checkout gluon sources and run 'make update'*/
        dir('gluon') {
          checkout scm: [$class: 'GitSCM',
            userRemoteConfigs: [[url: "${GLUON_URL}" ]],
            branches: [[name: "refs/tags/${GLUON_RELEASE_TAG}"]]], changelog: true, poll: false

          sh "make GLUON_SITEDIR=${workspace}/site GLUON_OUTPUTDIR=${workspace}/output update"
        }
    }

    /* build each target as a separate stage */
    for( target in targets ) {
      stage( target ) {
        dir('gluon') {
          sh "make clean GLUON_TARGET=${target} GLUON_SITEDIR=${workspace}/site GLUON_OUTPUTDIR=${workspace}/output"
          sh "make GLUON_RELEASE=\$(echo ${GLUON_RELEASE_TAG}|cut -c2-) GLUON_TARGET=${target} GLUON_SITEDIR=${workspace}/site GLUON_OUTPUTDIR=${workspace}/output V=99"
        }
      }
    }

    /* build libuecc, required for code signing */
    stage ("libuecc") {
        dir('libuecc') {
          checkout scm: [$class: 'GitSCM',
            userRemoteConfigs: [[url: 'git://git.universe-factory.net/libuecc']],
            branches: [[name: "master"]]], changelog: true, poll: false
        }

        dir('build/libuecc') {
          sh "rm -rf * && cmake -D CMAKE_INSTALL_PREFIX:PATH=${workspace}/lib ../../libuecc"
          sh "make install"
        }
    }

    /* build ecdsautils, required for code signing */
    stage ("ecdsautils") {
        dir('ecdsautils') {
          checkout scm: [$class: 'GitSCM',
            userRemoteConfigs: [[url: 'https://github.com/tcatm/ecdsautils.git']],
            branches: [[name: "master"]]], changelog: true, poll: false
        }

        dir('build/ecdsautils') {
          sh "rm -rf * && cmake -DCMAKE_INSTALL_PREFIX:PATH=${workspace}/lib ../../ecdsautils"
          sh "make install"
        }
    }

    /* create and sign manifest */
    stage ("manifest") {
        dir('gluon') {
          sh "make GLUON_SITEDIR=${workspace}/site GLUON_OUTPUTDIR=${workspace}/output GLUON_BRANCH=${GLUON_BRANCH} manifest"
          sh "LD_LIBRARY_PATH=${workspace}/lib/lib PATH=\$PATH:${workspace}/lib/bin ${workspace}/gluon/contrib/sign.sh ~/freifunk/fflauenburg ${workspace}/output/images/sysupgrade/${GLUON_BRANCH}.manifest"
        }
    }
}
