# Quack 🦆

![Rubber Duck](giant-rubber-duck.jpg?raw=true "Quack")
description: 'Run static code dependency scan using Black Duck Synopsys scan 
              for NPM and PIP using Synopsys 6.0. For details on properties 
              and the Black Duck Synopsis scan refer to their documentation
              here: https://blackducksoftware.github.io/synopsys-detect/6.0.0/.  
              Inputs prefaced with bd are referring to arguments in the 
              Black Duck scan.  Descriptions are shared from the current 
              documentation published.'
              
inputs:
  bd_url:
    description:    'URL of the Black Duck server.'
    required:       false
    default:        '${{ secrets.BLACKDUCK_URL }}'
  bd_api_token:
    description:    'The API token used to authenticate with the Black Duck Server.'
    required:       false
    default:        '${{ secrets.BLACKDUCK_API_TOKEN }}'
  detect_project_name:
    description:    'An override for the name to use for the Black Duck project. 
                    If not supplied, Detect will attempt to use the tools to figure 
                    out a reasonable project name. If that fails, the final part of 
                    the directory path where the inspection is taking place will be used.'
    required:       false
    default:        '${{ github.event.repository.name }}'
  detect_project_version_name: 
    description:    'An override for the version to use for the Black Duck project. 
                    If not supplied, Detect will attempt to use the tools to figure 
                    out a reasonable version name. If that fails, the current date 
                    will be used.'
    required:       false
    default:        '${GITHUB_REF##*/}' 
  detect_required_detector_types:
    description:    'The set of required detectors. If you want one or more detectors 
                    to be required (must be found to apply), use this property to 
                    specify the set of required detectors. If this property is set, 
                    and one (or more) of the given detectors is not found to apply, 
                    Detect will fail.'
    required:       false
    default:        'NPM'
  detect_included_detector_types:
    description:    'By default, all tools will be included. If you want to include 
                    only specific tools, specify the ones to include here. Exclusion 
                    rules always win. If you want to limit Detect to a subset of its 
                    detectors, use this property to specify that subset.'    
    required:       false
    default:        'NPM,PIP'
  detect_tools:
    description:    'The tools Detect should allow in a comma-separated list. Tools 
                    in this list (as long as they are not also in the excluded list) 
                    will be allowed to run if all criteria of the tool are met. 
                    Exclusion rules always win. This property and detect.tools.excluded 
                    provide control over which tools Detect runs.'
    required:       false
    default:        'DETECTOR'
  detect_detector_search_depth:
    description:    'Depth of subdirectories within the source directory to which 
                    Detect will search for files that indicate whether a detector 
                    applies. A value of 0 (the default) tells Detect not to search 
                    any subdirectories, a value of 1 tells Detect to search first-level 
                    subdirectories, etc.'
    required:       false
    default:        '15'
  detect_npm_include_dev_dependencies:
    description:    'Set this value to false if you would like to exclude your dev 
                    dependencies when ran.'
    required:       false
    default:        'false'