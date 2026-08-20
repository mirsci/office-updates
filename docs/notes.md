Goal
Implement a multi-tenant common dev environment, to deploy SLM for inference, model evals flows, fine-tuning processes

Scope
- Use both OTB and fine-tuned Gemma 4 E4B for inference, baselining and model evaluations
- Primary environment: shared Linux machine + remote GPU machine
- Secondary targets: macOS (dev approximation), iOS (via Apple hardware)

Use cases (technical)
1. Each dev can run parallel/multi-user model inference and Gemma 4 LLM evals in developer mode
2. A developer can access shared model artifacts and eval datasets without per-dev download
3. A developer can reproduce a baseline eval run and compare against team-shared references
4. A developer can run inference against multiple fine-tuned model versions and compare their behaviour
5. A developer can run data synthesis workflows independently of the inference server
6. A developer can clean and reset their local environment without affecting teammates
7. The team can validate GPU readiness before scheduling expensive inference or eval runs
