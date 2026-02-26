<!-- BUILD_PLAN_TEMPLATE.md
     Claude reads this template when helping users create a new project's BUILD_PLAN.md.
     Each section includes guidance on what to fill in and notes on CI/CD compatibility. -->

<project_specification>
  <project_name>
    <!-- A short, descriptive name for the project.
         Example: "Canopy - Full Featured Project Management Application" -->
  </project_name>

  <overview>
    <!-- Describe in 3-5 paragraphs:
         - What the application does and who it's for
         - Key user workflows and value proposition
         - High-level architecture choice (monorepo vs separate repos, monolith vs microservices)
         - How frontend and backend communicate (e.g., REST API via env var, GraphQL, etc.)

         ARCHITECTURE NOTE: Structure as a monorepo with these directories:
           shared/    - Shared types/schemas (source of truth for API contract)
           frontend/  - Client application
           backend/   - Server-side handlers
           infrastructure/  - IaC definitions (CDK, Terraform, etc.)

         CI/CD COMPATIBILITY:
         The existing pipeline natively supports:
           - React + Vite frontend (build output in dist/)
           - AWS Lambda + API Gateway backend
           - DynamoDB for persistence
           - CDK for infrastructure
           - deploy-preview.yml deploys frontend to S3 + CloudFront
           - deploy-infrastructure.yml runs cdk deploy for backend resources

         If you choose a different stack (e.g., Next.js, PostgreSQL, ECS), note which
         workflows or IAM policies would need adjustment. -->
  </overview>

  <technology_stack>
    <frontend_application>
      <!-- List each choice on its own line:
           - Framework (React, Vue, Svelte, etc.)
           - Build tool (Vite, Next.js, etc.)
           - Styling (Tailwind CSS, CSS Modules, styled-components, etc.)
           - Routing library
           - State management approach
           - Key UI libraries (drag-and-drop, charts, date handling, icons, etc.) -->
    </frontend_application>

    <data_layer>
      <!-- - Database (DynamoDB, PostgreSQL, etc.)
           - API style (REST via Lambda + API Gateway, GraphQL via AppSync, etc.)
           - API client approach (fetch, axios, React Query, Apollo, etc.)
           - Search strategy (server-side queries, client-side indexing, OpenSearch, etc.)
           - Shared contract mechanism (Zod schemas, TypeScript interfaces, OpenAPI, etc.)

           CI/CD NOTE: The default pipeline assumes Lambda + API Gateway + DynamoDB.
           Other databases (RDS, Aurora) require VPC configuration in CDK.
           Non-Lambda compute (ECS, App Runner) requires new deploy workflows. -->
    </data_layer>

    <build_output>
      <!-- How each layer builds:
           - Frontend: e.g., "npm run build in frontend/ produces dist/"
           - Backend: e.g., "esbuild bundles Lambda handlers (handled by CDK)"
           - Infrastructure: e.g., "cdk synth produces CloudFormation template"

           CI/CD NOTE: deploy-preview.yml expects a dist/ folder from the frontend build.
           If your build output differs, update the find pattern in that workflow. -->
    </build_output>
  </technology_stack>

  <infrastructure>
    <cdk_stack>
      <!-- Define the AWS resources the CDK stack should create:
           - API Gateway (HTTP API or REST API)
           - Lambda functions (list handlers and their routes)
           - Database tables (DynamoDB tables, GSIs, or RDS instances)
           - S3 buckets (if needed beyond the preview bucket)
           - IAM roles and policies
           - Any other AWS resources

           The agent will write this as infrastructure/lib/{project}-stack.ts.
           Include enough detail for the agent to implement the full stack. -->
    </cdk_stack>

    <cdk_testing>
      <!-- Describe what CDK tests should verify:
           - Resource counts (e.g., "stack creates 1 HTTP API, 5 Lambda functions")
           - Resource properties (e.g., "Lambda runtime is nodejs20.x")
           - IAM policy assertions
           Tests go in infrastructure/test/ and use aws-cdk-lib/assertions. -->
    </cdk_testing>

    <database_design>
      <!-- For DynamoDB: describe table name, partition key, sort key, GSIs.
           For SQL databases: describe tables, columns, relationships, indexes.
           Include access patterns — how will data be queried?

           DynamoDB SINGLE-TABLE TIP: If using single-table design, define the
           PK/SK patterns for each entity (e.g., PK=PROJECT#id, SK=ISSUE#id). -->
    </database_design>
  </infrastructure>

  <api_contract>
    <!-- Define the shared types/schemas that both frontend and backend use.
         This is the most important section for preventing contract mismatches.

         Include:
         - Directory structure for shared/ package
         - Schema definitions for each entity (fields, types, validation rules)
         - Request/response types for each API endpoint
         - Endpoint route map (method, path, request type, response type)

         Example structure:
           shared/
             src/
               schemas/
                 project.ts    - ProjectSchema, CreateProjectRequest, etc.
                 issue.ts      - IssueSchema, CreateIssueRequest, etc.
               types.ts        - Re-exports all types
               index.ts        - Package entry point -->
  </api_contract>

  <core_data_entities>
    <!-- Define each entity the application manages. For each entity include:
         - Field names and types
         - Required vs optional fields
         - Relationships to other entities
         - Default values
         - Validation rules

         Be thorough — the agent uses this to generate schemas, database access
         patterns, API handlers, and UI forms. -->
  </core_data_entities>

  <pages_and_interfaces>
    <!-- Describe every page/view in the application. For each page include:
         - Layout structure (header, sidebar, main content areas)
         - Components and their behavior
         - User interactions (click, drag, keyboard shortcuts)
         - Empty states
         - Loading states
         - Error states

         Be specific about UI behavior — the agent builds exactly what you describe.
         Vague descriptions produce vague UIs. -->
  </pages_and_interfaces>

  <core_functionality>
    <!-- Describe the key features and workflows. Group by domain area.
         For each feature include:
         - What the user can do
         - What happens on the backend
         - Edge cases to handle
         - How it integrates with other features -->
  </core_functionality>

  <aesthetic_guidelines>
    <!-- Define the visual design system:
         - Color palette (primary, background, text, status colors with hex values)
         - Typography (font families, sizes, line heights)
         - Spacing scale
         - Border radii and shadows
         - Component styling (buttons, inputs, cards, modals, badges)
         - Animation and transition guidelines
         - Icon style
         - Accessibility requirements (contrast ratios, keyboard navigation, ARIA)

         TIP: Be specific with hex values and pixel sizes. The agent follows
         exact specifications better than vague descriptions like "modern" or "clean". -->
  </aesthetic_guidelines>

  <final_integration_test>
    <!-- Define 8-12 end-to-end user scenarios that exercise the full application.
         Each scenario should include:
         - Description of the workflow
         - Step-by-step actions
         - Expected outcomes at each step

         These become the agent's acceptance criteria and guide test writing. -->
  </final_integration_test>

  <success_criteria>
    <!-- Define what "done" looks like across dimensions:
         - Functionality: which features must work
         - User experience: responsiveness, transitions, feedback
         - Technical quality: TypeScript strict mode, no console errors, tests pass
         - Visual design: adherence to aesthetic guidelines
         - Build: clean build with no warnings -->
  </success_criteria>

  <build_output>
    <!-- Specify the build command and output:
         - Build command (e.g., "npm run build")
         - Output directory (e.g., "dist/")
         - What the output should contain (index.html, JS bundles, assets)

         CI/CD NOTE: deploy-preview.yml looks for dist/ in workspace subdirectories.
         If your output directory is different, note it here. -->
  </build_output>

  <key_implementation_notes>
    <!-- Capture important technical decisions and constraints:
         - Critical implementation paths the agent should follow
         - Recommended implementation order (which layers to build first)
         - Monorepo structure and build dependencies
         - Performance considerations
         - Testing strategy (unit, integration, e2e; which tools to use)

         AGENT BUILD ORDER: The agent builds in phases:
           Phase 1: Shared types + Frontend scaffold (React + Vite + Tailwind)
           Phase 2a: CDK infrastructure + stub Lambda handlers → commit & push
           Phase 2b: Wait for CI/CD deployment, then implement full handlers
           Phase 3: Wire VITE_API_URL into frontend, connect to live API
           Phase 4: Polish UI, write tests, final commit

         Your implementation notes should align with this phased approach. -->
  </key_implementation_notes>

  <sample_data>
    <!-- Provide realistic sample data the agent should seed into the application.
         This helps with testing and makes demos look realistic.
         Include sample values for each entity type. -->
  </sample_data>
</project_specification>
