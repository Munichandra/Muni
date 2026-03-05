# Salesforce DX Project: Next Steps

Now that you’ve created a Salesforce DX project, what’s next? Here are some documentation resources to get you started.

## How Do You Plan to Deploy Your Changes?

Do you want to deploy a set of changes, or create a self-contained application? Choose a [development model](https://developer.salesforce.com/tools/vscode/en/user-guide/development-models).

## Configure Your Salesforce DX Project

The `sfdx-project.json` file contains useful configuration information for your project. See [Salesforce DX Project Configuration](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_ws_config.htm) in the _Salesforce DX Developer Guide_ for details about this file.

## Read All About It

- [Salesforce Extensions Documentation](https://developer.salesforce.com/tools/vscode/)
- [Salesforce CLI Setup Guide](https://developer.salesforce.com/docs/atlas.en-us.sfdx_setup.meta/sfdx_setup/sfdx_setup_intro.htm)
- [Salesforce DX Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_intro.htm)

stages:
    - validate
    - test
    - deploy
    - postDeploy

image: registry.gitlab.com/syngenta-latam/ecomm:latest

MR Build Validation IST:
    rules:
        - if: $CI_MERGE_REQUEST_TARGET_BRANCH_NAME == "release/IST"
    stage: validate
    tags:
        - $CI_MERGE_REQUEST_TARGET_BRANCH_NAME
    script:
        - echo "MR build validation" 

        - echo "fetching source and target branches from repo"
        - ScriptforMR

MR Build Validation UAT:
    rules:
        - if: $CI_MERGE_REQUEST_TARGET_BRANCH_NAME == "release/UAT"
    stage: validate
    tags:
        - $CI_MERGE_REQUEST_TARGET_BRANCH_NAME
    script:
        - echo "MR build validation" 

        - echo "fetching source and target branches from repo"
        - ScriptforMR


MR Build Validation Production:
    rules:
        - if: $CI_MERGE_REQUEST_TARGET_BRANCH_NAME == "master"
    stage: validate
    tags:
        - $CI_MERGE_REQUEST_TARGET_BRANCH_NAME
    script:
        - echo "MR build validation" 

        - echo "fetching source and target branches from repo"        
        - ScriptforMR


Branch Build Validation for IST:
    only:
        - release/IST
    stage: test
    tags:
        - release/IST
    script:
        - echo "build validation"
        - runsgdpackageforcommits
        - sfdx force:source:deploy --checkonly -p "srcDiff/force-app" -u aliasuser
        #- sfdx force:source:deploy --checkonly -p srcDiff/force-app -u devgsa  
    
Branch Build Validation UAT:
    only:
        - release/UAT
    stage: test
    tags:
        - release/UAT
    script:
        - echo "build validation"
        - runsgdpackageforcommits
        - sfdx force:source:deploy --checkonly -p "srcDiff/force-app" -u aliasuser
        #- sfdx force:source:deploy --checkonly -p srcDiff/force-app -u devgsa  

Branch Build Validation:
    only:
        - master
    stage: test
    tags:
        - master
    script:
        - echo "build validation"
        - runsgdpackageforcommits
        - sfdx force:source:deploy --checkonly -p "srcDiff/force-app" --testlevel=RunLocalTests -u aliasuser
        #- sfdx force:source:deploy --checkonly -p srcDiff/force-app -u devgsa  

Branch Deployment IST:
    only:
        - release/IST
    stage: deploy
    tags:
        - release/IST
    script:
        - echo "build deployement"
        - runsgdpackageforcommits
        - sfdx force:source:deploy -p "srcDiff/force-app" -u aliasuser
        #- sfdx force:source:deploy --checkonly -p srcDiff/force-app -u devgsa  
    
Branch Deployment UAT:
    only:
        - release/UAT
    stage: deploy
    tags:
        - release/UAT
    script:
        - echo "build deployement"
        - runsgdpackageforcommits
        - sfdx force:source:deploy -p "srcDiff/force-app" -u aliasuser
        #- sfdx force:source:deploy --checkonly -p srcDiff/force-app -u devgsa  

Branch Deployment:
    only:
        - master
    stage: deploy
    tags:
        - master
    script:
        - echo "build deployement"
        - runsgdpackageforcommits
        - sfdx force:source:deploy -p "srcDiff/force-app" --testlevel=RunLocalTests -u aliasuser
        #- sfdx force:source:deploy --checkonly -p srcDiff/force-app -u devgsa  

master merge to IST:
    only:
        - master
    stage: postDeploy
    tags:
        - release/IST
    script:
        - echo "master merge to IST"
        - git config remote.origin.fetch '+refs/heads/*:refs/remotes/origin/*'
        - git fetch origin master
        - git fetch origin release/IST
        - git checkout release/IST
        - git config --global user.email "devopsuser@invalid.com"
        - git config --global user.name "devopsuser"
        - git merge --no-ff --no-commit origin/master
        - git reset HEAD .gitlab-ci.yml
        - git checkout -- .gitlab-ci.yml
        - git commit -m "Merge from master to IST"
        - git push https://${USER_TO_MERGE}:${CI_PERSONAL_TOKEN}@gitlab.com/syngenta-latam/ecomm.git release/IST
    allow_failure: true

master merge to UAT:
    only:
        - master
    stage: postDeploy
    tags:
        - release/UAT
    script:
        - echo "build master merge to UAT"
        - git config remote.origin.fetch '+refs/heads/*:refs/remotes/origin/*'
        - git fetch origin master
        - git fetch origin release/UAT
        - git checkout release/UAT
        - git config --global user.email "devopsuser@invalid.com"
        - git config --global user.name "devopsuser"
        - git merge --no-ff --no-commit origin/master
        - git reset HEAD .gitlab-ci.yml
        - git checkout -- .gitlab-ci.yml
        - git commit -m "Merge from master to UAT"
        - git push https://${USER_TO_MERGE}:${CI_PERSONAL_TOKEN}@gitlab.com/syngenta-latam/ecomm.git release/UAT
    allow_failure: true


.helpers: &helpers |

    # Function to find last successful build commit id.
    # No arguments.

    function ScriptforMR() {
         runsgdpackage
        skipTheJobIfNoMetadataChanges
        echo "Target branch $CI_MERGE_REQUEST_TARGET_BRANCH_NAME"
        update_org_creds $CI_MERGE_REQUEST_TARGET_BRANCH_NAME
        echo " org user-name $USER_NAME"
        cat srcDiff/package/package.xml
        cat srcDiff/destructiveChanges/destructiveChanges.xml
        # decrypt private keyENCRYPTION_KEY  ENCRYPTION_IV USER_NAME CONSUMER_KEY
        keyDecriptAsPerENV $CI_MERGE_REQUEST_TARGET_BRANCH_NAME
        # authenticate to enviroment using SFDX JWT flow
        authenticateAsPerENV $CI_MERGE_REQUEST_TARGET_BRANCH_NAME
        # change directory to folder containing copied source files that included in the pull request
        sfdx force:source:deploy --checkonly --testlevel=RunLocalTests -p "srcDiff/force-app" -u aliasuser
        #- sfdx force:source:deploy --checkonly -p srcDiff/force-app -u devgsa 
    }

    function getLastSuccessfulBuildCommit() {
        last_successful_build_commit=$CI_COMMIT_SHA
        pipeline_ids=$(curl -s --header "PRIVATE-TOKEN: $GITLAB_TOKEN" "https://gitlab.com/api/v4/projects/$CI_PROJECT_ID/pipelines?ref=$CI_COMMIT_BRANCH&per_page=$GIT_HISTORY_LIMIT" |  jq '.[]' | jq .id)
        
        for pipeline_id in $pipeline_ids; do
                build_status=$(curl -s --header "PRIVATE-TOKEN: $GITLAB_TOKEN" "https://gitlab.com/api/v4/projects/$CI_PROJECT_ID/pipelines/$pipeline_id" | jq .status)

                if [ $build_status = \"success\" ]
                then
                    commit_id_sha=$(curl -s --header "PRIVATE-TOKEN: $GITLAB_TOKEN" "https://gitlab.com/api/v4/projects/$CI_PROJECT_ID/pipelines/$pipeline_id" | jq .sha)
                    commit_id=$(echo $commit_id_sha | tr -d '"')
                    last_successful_build_commit=$commit_id
                    echo "last commit from script $last_successful_build_commit"
                    break
                fi
        done
        # echo "$last_successful_build_commit has successful build"
        export last_successful_build_commit
    }


    function update_org_creds() {
        if [ $1 = "master" ]
        then
            ENCRYPTION_KEY=$ENCRYPTION_KEY_PRD
            ENCRYPTION_IV=$ENCRYPTION_IV_PRD
            USER_NAME=$USER_NAME_PRD
            CONSUMER_KEY=$CONSUMER_KEY_PRD
            export ENCRYPTION_KEY  ENCRYPTION_IV USER_NAME CONSUMER_KEY
        elif [ $1 = "release/IST" ]
        then
            ENCRYPTION_KEY=$ENCRYPTION_KEY_IST
            ENCRYPTION_IV=$ENCRYPTION_IV_IST
            USER_NAME=$USER_NAME_IST
            CONSUMER_KEY=$CONSUMER_KEY_IST
            export ENCRYPTION_KEY  ENCRYPTION_IV USER_NAME CONSUMER_KEY
        elif [ $1 = "release/UAT" ]
        then
            ENCRYPTION_KEY=$ENCRYPTION_KEY_UAT
            ENCRYPTION_IV=$ENCRYPTION_IV_UAT
            USER_NAME=$USER_NAME_UAT
            CONSUMER_KEY=$CONSUMER_KEY_UAT
            export ENCRYPTION_KEY  ENCRYPTION_IV USER_NAME CONSUMER_KEY
        else
            echo "No environement found"
        fi
    }

    function keyDecriptAsPerENV() {
        if [ $1 = "master" ]
        then
            openssl enc -d -aes-256-cbc -in build/prod/server.key.enc -out build/prod/server.key -K $ENCRYPTION_KEY -iv $ENCRYPTION_IV
        else
            openssl enc -d -aes-256-cbc -in build/server.key.enc -out build/server.key -K $ENCRYPTION_KEY -iv $ENCRYPTION_IV
        fi
    }

    function authenticateAsPerENV() {
        if [ $1 = "master" ]
        then
            sfdx force:auth:jwt:grant -u $USER_NAME -a aliasuser -f build/prod/server.key -i $CONSUMER_KEY -s --instanceurl https://login.salesforce.com
        else
            sfdx force:auth:jwt:grant -u $USER_NAME -a aliasuser -f build/server.key -i $CONSUMER_KEY -s --instanceurl https://test.salesforce.com
        fi
    }

    function installCodeFormatLintJest() {
      npm install --save-dev --save--exact prettier prettier-plugin-apex
      npm install --save-dev --save--exact jest @prettier/plugin-xml eslint @salesforce/eslint-config-lwc @salesforce/eslint-plugin-aura @babel/eslint-parser @lwc/eslint-plugin-lwc @salesforce/sfdx-lwc-jest@0.10.4 eslint-config-prettier @babel/core @babel/eslint-parser @lwc/eslint-plugin-lwc
    }

    function skipTheJobIfNoMetadataChanges() {
        if [ -d "srcDiff/force-app" ]
        then
            echo "Metadata changes are there"
        else
            echo "No metadata changes are found"
            exit 0
        fi
    }

    function runsgdpackage() {
        git config remote.origin.fetch '+refs/heads/*:refs/remotes/origin/*'
        git fetch origin $CI_MERGE_REQUEST_TARGET_BRANCH_NAME
        git fetch origin $CI_MERGE_REQUEST_SOURCE_BRANCH_NAME
        # Create src diff folder
        mkdir srcDiff
        echo "getting delta between branches"
        sgd --from origin/$CI_MERGE_REQUEST_TARGET_BRANCH_NAME --to origin/$CI_MERGE_REQUEST_SOURCE_BRANCH_NAME --repo . --output srcDiff --generate-delta -a $SF_API_VERSION
        ls -lR srcDiff
    }

    function runsgdpackageforcommits() {
        git config remote.origin.fetch '+refs/heads/*:refs/remotes/origin/*'
        git fetch origin $CI_COMMIT_BRANCH
        getLastSuccessfulBuildCommit
        echo "$last_successful_build_commit is the last successful build commit"
        echo "$CI_COMMIT_SHA is latest commit"
        mkdir srcDiff
        # generating package and destructive package using SFDX git delta package
        sgd --from $last_successful_build_commit --to $CI_COMMIT_SHA --repo . --output srcDiff --generate-delta -a $SF_API_VERSION
        # List srcDiff files 
        ls -lR srcDiff
        skipTheJobIfNoMetadataChanges
 
        echo "Target branch $CI_COMMIT_BRANCH"
        update_org_creds $CI_COMMIT_BRANCH
        echo " org user-name $USER_NAME" 
        # print package details for reference
        cat srcDiff/package/package.xml
        cat srcDiff/destructiveChanges/destructiveChanges.xml
        keyDecriptAsPerENV $CI_COMMIT_BRANCH
        authenticateAsPerENV $CI_COMMIT_BRANCH
    }

    function skipLintIfNoLwcChanges() {
        if [ -d "srcDiff/force-app/main/default/lwc" ]
        then
            echo "LWC changes are there"
            npm run lint
        else
            echo "No LWC changes are found"
        fi
    }

    function apexTestValidation() {
         failure=$(cat out.json  | grep "failing" | cut -d ':' -f2 $line | xargs)
         failure_test_cases="${failure::-1}"
         echo "failed test cases $failure_test_cases"
         if [ $failure_test_cases -gt 0 ]
         then
            echo "Seems few test cases are failed"
            exit 1
         fi

         org_wide_coverage=$(cat out.json  | grep "orgWideCoverage" | cut -d ':' -f2 $line | xargs | cut -d ' ' -f2 $line)
         echo "Org wide test coverage is $org_wide_coverage"
         coverage="${org_wide_coverage::-1}"
         if [ $coverage -lt $ORG_WIDE_TEST_COVERAGE ]
         then
            echo "Org wide coverage is below 85%, marking build failure"
            exit 1
         fi
    }

    function getClaytonBuildStatus() {
      while true
      do
          build_status=$(curl -s --header "PRIVATE-TOKEN:$GITLAB_TOKEN" "https://gitlab.com/api/v4/projects/$CI_PROJECT_ID/repository/commits/$CI_COMMIT_SHA/statuses" | jq -r '. | map([.name, .status |tostring ] | join("|")) | join("\n")' | grep Clayton | cut -d '|' -f2 $line)
          echo $build_status
          if [ $build_status = "success" ] || [ $build_status = "failed" ];
          then
                  clayton_build_status=$build_status
                  break
          else
                  echo "Waiting for clayton build status"
          fi
      done

      export clayton_build_status
    }
    

before_script:
    - *helpers
    

- [Salesforce CLI Command Reference](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference.htm)
