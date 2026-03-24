##################################################
# Frontend overview

platform.orbitai.fun                Production
platform-stg.orbitai.fun            Staging
platform-stg.orbitai.fun            Development

Deploy: Vercel

##################################################
# Gateway overview

platform-gateway.orbitai.fun        Production  gateway 41003
platform-gateway-stg.orbitai.fun    Staging     gateway 41002
platform-gateway-dev.orbitai.fun    Development gateway 41001

Deploy: VPS
CI/CD:  Github Action
Type:   Docker Image
