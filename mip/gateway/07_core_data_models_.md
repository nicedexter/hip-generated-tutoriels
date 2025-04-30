# Chapter 7: Core Data Models

Welcome to the final chapter! In [Chapter 6: Result Handlers (Engine Connectors)](06_result_handlers__engine_connectors_.md), we saw how connectors use handlers to translate raw results from backend engines into *standardized formats* like tables or charts. But what are these standard formats, exactly? And what about other key concepts in the `gateway` like "Users", "Domains", or "Experiments"? How are *they* consistently represented?

That's where **Core Data Models** come in.

## What's the Big Idea? (Motivation)

Imagine building a complex machine, like a car. You need blueprints! These blueprints define exactly what each part looks like and what its specifications are (e.g., "This is a Spark Plug, it has this size thread, this gap..."). Everyone involved in building the car – the engine team, the electronics team, the assembly line – uses the same blueprints. This ensures all the parts fit together and work correctly.

In the `gateway` project, our **Core Data Models** are like those **blueprints**. They are TypeScript classes that define the standard structure for all the important concepts, or "nouns," used within the system.

*   **Why do we need them?**
    *   **Consistency:** Whether data about available datasets (a `Domain`) comes from an Exareme backend or a DataShield backend, the [Engine Connector](05_engine___connectors_.md) translates it into the *same* standard `Domain` structure.
    *   **Clear API:** The [GraphQL API Layer](01_graphql_api_layer_.md) uses these models (decorated with `@ObjectType`) to define its public "menu" (schema). Clients know exactly what structure to expect for a `Domain`, `Experiment`, or `User`.
    *   **Predictable Code:** Services (like `ExperimentsService` or `UsersService`) and database interactions (using TypeORM) all work with these same standard structures.

Our goal in this chapter is to understand what these core blueprints look like and why they are the foundation for consistent data handling throughout the `gateway`.

## Key Ingredients (The Blueprints)

Let's look at the most important blueprints, our Core Data Models:

1.  **`User` (`user.model.ts`)**:
    *   **What it represents:** The person currently logged into the `gateway`.
    *   **Key fields:** `id` (unique identifier), `username`, `fullname`, `email`, and gateway-specific fields like `agreeNDA` and `refreshToken` (hash) as discussed in [Chapter 3: User Service & Entity](03_user_service___entity_.md).
    *   **Where it's used:** Authentication checks ([Chapter 2: Authentication & Authorization](02_authentication___authorization_.md)), storing gateway-specific user info, associating experiments with their creator.

2.  **`Domain` (`domain.model.ts`)**:
    *   **What it represents:** A specific data universe or context, like "Public Health Study X" or "Genomics Data v2". It acts as a container for related data elements.
    *   **Key fields:** `id` (unique identifier), `label` (display name), `datasets` (list of actual data tables/files), `variables` (list of data columns/features available), `groups` (folders for organizing variables).
    *   **Where it's used:** Fetched by [Engine Connectors](05_engine___connectors_.md) from backends, displayed to users so they can choose which data context to work in, referenced by `Experiment`s.

3.  **`Variable` (`variable.model.ts`)**:
    *   **What it represents:** A single data column or feature within a `Domain`, like "Age", "Blood_Pressure", or "Gene_Expression_Level".
    *   **Key fields:** `id`, `label`, `type` (e.g., 'numerical', 'categorical'), `enumerations` (possible values for categorical variables), `groups` (which folders it belongs to).
    *   **Where it's used:** Displayed within a `Domain`, selected by users when defining an `Experiment`.

4.  **`Group` (`group.model.ts`)**:
    *   **What it represents:** A folder or category used to organize `Variable`s within a `Domain`, making it easier to navigate large datasets (e.g., a "Demographics" group containing "Age" and "Gender" variables).
    *   **Key fields:** `id`, `label`, `variables` (list of variable IDs in this group), `groups` (list of subgroup IDs).
    *   **Where it's used:** Structuring the display of `Variable`s within a `Domain`.

5.  **`Dataset` (`dataset.model.ts`)**:
    *   **What it represents:** A specific table, file, or cohort of data within a `Domain`. For example, a `Domain` called "Medical Study" might have `Dataset`s like "Baseline_Data" and "FollowUp_Data".
    *   **Key fields:** `id`, `label`.
    *   **Where it's used:** Listed within a `Domain`, selected by users when defining an `Experiment`.

6.  **`Algorithm` (`algorithm.model.ts`)**:
    *   **What it represents:** A computational method or statistical analysis that can be run, like "Linear Regression", "T-Test", or "ANOVA".
    *   **Key fields:** `id`, `label`, `type` (e.g., 'statistical_model'), `parameters` (list of settings the algorithm requires).
    *   **Where it's used:** Fetched by [Engine Connectors](05_engine___connectors_.md) from backends, listed for users to choose from when creating an `Experiment`.

7.  **`Parameter` (`base-parameter.model.ts` and variants like `number-parameter.model.ts`)**:
    *   **What it represents:** A specific setting or option for an `Algorithm`, like the "Significance Level (alpha)" for a T-Test or the "Number of Clusters (k)" for K-Means.
    *   **Key fields:** `id`, `label`, `type` (e.g., 'number', 'enum', 'variable'), possible `defaultValue`, `min`/`max` values, `enumValues`.
    *   **Where it's used:** Defined within an `Algorithm`, presented to the user to fill in when creating an `Experiment`.

8.  **`Experiment` (`experiment.model.ts`)**:
    *   **What it represents:** A record of a specific run of an `Algorithm` on particular data with specific `Parameter` values. Think of it as a single entry in your digital lab notebook.
    *   **Key fields:** `id`, `name`, `status` (e.g., PENDING, SUCCESS, ERROR), `author` (the `User` who ran it), `createdAt`, `finishedAt`, `domain`, `datasets`, `variables`, `algorithm` (name and parameters used), `results`.
    *   **Where it's used:** Created via the `createExperiment` mutation ([Chapter 4: Experiment Management](04_experiment_management_.md)), stored in the gateway database (often), retrieved to show status and results.

9.  **`ResultUnion` (`result-union.model.ts`) & Concrete Types (`TableResult`, `BarChartResult`, etc.)**:
    *   **What it represents:** The standardized formats for the outputs of an `Experiment`, as produced by the [Result Handlers (Engine Connectors)](06_result_handlers__engine_connectors_.md).
    *   **Key fields (Examples):**
        *   `TableResult`: `name`, `headers` (column definitions), `data` (2D array of strings).
        *   `BarChartResult`: `name`, `xAxis`/`yAxis` labels, `barValues`.
        *   `AlertResult`: `message`, `type` (e.g., 'info', 'warning', 'error').
    *   **Where it's used:** Populated within the `Experiment.results` array by Result Handlers, defined in the GraphQL schema so the frontend knows how to render different result types.

## How They Are Used (Ensuring Consistency)

These models act as the common language spoken throughout the gateway.

*   **Connectors:** When an `ExaremeConnector` fetches data about pathologies, it transforms that Exareme-specific JSON into the standard `Domain`, `Variable`, `Group`, and `Dataset` model objects.
*   **GraphQL API:** The `ExperimentsResolver` uses the `Experiment` model (decorated with `@ObjectType`) to define what an `Experiment` looks like in the GraphQL schema. When a client queries for `experimentList`, the resolver returns data structured according to this `Experiment` model blueprint.
*   **Services & Database:** The `ExperimentsService` uses the `Experiment` model (also decorated as a TypeORM `@Entity`) to interact with the gateway's database, saving and retrieving experiment records that match the defined structure.

This ensures that data keeps its shape and meaning as it flows between the backend engine, the gateway's internal logic, the database, and the frontend client.

## A Peek Inside the Blueprints (Code Examples)

Let's look at simplified versions of some core model classes. Note the decorators:
*   `@ObjectType()`: Makes the class and its fields available in the [GraphQL API Layer](01_graphql_api_layer_.md).
*   `@Field()`: Exposes a specific property in the GraphQL API.
*   `@Entity()` / `@Column()` / `@PrimaryGeneratedColumn()`: (From TypeORM) Map the class and its properties to a database table and columns, used by services like [User Service & Entity](03_user_service___entity_.md) and [Experiment Management](04_experiment_management_.md).

**Example 1: `Domain` Model (`domain.model.ts`)**

```typescript
// File: api/src/engine/models/domain.model.ts (Simplified)
import { Field, ObjectType } from '@nestjs/graphql';
import { Dataset } from './dataset.model';
import { Group } from './group.model';
import { Variable } from './variable.model';
// BaseModel might contain common fields like id, label

@ObjectType() // Make this available in GraphQL schema
export class Domain /* extends BaseModel */ {
  @Field() // Expose 'id' in GraphQL
  id: string;

  @Field() // Expose 'label' in GraphQL
  label: string;

  // Define relationships to other models
  @Field(() => [Group]) // It has an array of Group objects
  groups: Group[];

  @Field(() => [Variable]) // It has an array of Variable objects
  variables: Variable[];

  @Field(() => [Dataset]) // It has an array of Dataset objects
  datasets: Dataset[];
}
```
*Explanation:* This defines the blueprint for a `Domain`. `@ObjectType` and `@Field` expose it via GraphQL. It specifies that a `Domain` has an `id`, a `label`, and lists of related `Group`, `Variable`, and `Dataset` objects, linking the different blueprints together.

**Example 2: `Experiment` Model (`experiment.model.ts`)**

```typescript
// File: api/src/engine/models/experiment/experiment.model.ts (Simplified)
import { Field, ObjectType, registerEnumType } from '@nestjs/graphql';
import { Column, Entity, PrimaryGeneratedColumn } from 'typeorm'; // For Database mapping
import { ResultUnion } from '../result/common/result-union.model'; // Standard results
import { Author } from './author.model'; // Simplified Author info

// Define possible statuses
export enum ExperimentStatus { PENDING = 'pending', SUCCESS = 'success', ERROR = 'error' }
registerEnumType(ExperimentStatus, { name: 'ExperimentStatus' });

@Entity() // Map this class to a database table named 'experiment'
@ObjectType() // Make this available in GraphQL
export class Experiment {
  @PrimaryGeneratedColumn('uuid') // Auto-generate ID in DB
  @Field() // Expose ID in GraphQL
  id: string;

  @Column() // Map 'name' to a DB column
  @Field() // Expose 'name' in GraphQL
  name: string;

  @Field(() => ExperimentStatus) // Expose status
  @Column({ type: 'enum', enum: ExperimentStatus }) // Map status to DB enum
  status?: ExperimentStatus;

  @Field(() => Author) // Expose author info
  @Column('jsonb') // Store author object as JSON in DB
  author?: Author;

  @Field(() => [String]) // Expose list of dataset IDs used
  @Column('text', { array: true }) // Store as text array in DB
  datasets: string[];

  // Field for Algorithm details (simplified)
  @Field()
  @Column('jsonb')
  algorithm: { name: string; parameters: { name: string; value: string }[] };

  // Array holding the standardized results!
  @Field(() => [ResultUnion]) // Use the Union type from Chapter 6
  @Column('jsonb') // Store results array as JSON in DB
  results?: Array<typeof ResultUnion>;
}
```
*Explanation:* This defines the `Experiment` blueprint. It uses `@Entity` and `@Column` for database storage (managed by `ExperimentsService`) and `@ObjectType`/`@Field` for the GraphQL API. Notice how it includes references to other models (`Author`), lists of identifiers (`datasets`), and importantly, the `results` field which holds an array of standardized `ResultUnion` objects.

**Example 3: `TableResult` Model (`table-result.model.ts`)**

```typescript
// File: api/src/engine/models/result/table-result.model.ts (Simplified)
import { Field, ObjectType } from '@nestjs/graphql';
import { Header } from './common/header.model'; // Blueprint for a column header
// Result might be a base class with common fields

@ObjectType() // Part of the ResultUnion, available in GraphQL
export class TableResult /* extends Result */ {
  @Field() // Expose name in GraphQL
  name: string; // e.g., "Model Coefficients"

  @Field(() => [Header]) // Expose column headers
  headers: Header[]; // Array of Header objects (e.g., {name: 'Variable', type: 'string'})

  @Field(() => [[String]]) // Expose the data grid
  data: string[][]; // A 2D array of strings representing rows and cells
}
```
*Explanation:* This blueprint defines the standard structure for a simple table result. It has a `name`, an array of `Header` objects (each defining a column), and a `data` field holding the actual table content as a 2D array of strings. The [Result Handlers](06_result_handlers__engine_connectors_.md) inside connectors are responsible for translating raw backend output into this specific structure.

**Visualizing the Flow:**

Here's how these models ensure consistency when fetching data (like `Domain`s) from a backend:

```mermaid
graph LR
    BackendAPI[Backend API (e.g., Exareme)] -- Raw JSON --> Connector(Engine Connector);
    Connector -- Transforms --> ModelObj[Core Model Object (e.g., Domain)];
    ModelObj -- Used by --> Resolver(GraphQL Resolver);
    ModelObj -- Saved/Loaded by --> Service(Service);
    Service -- Interacts with --> Database[(Gateway DB)];
    Resolver -- Defines schema & returns --> GraphQLResponse[GraphQL API Response];
```
*Explanation:* The Connector translates raw backend data into a standard Core Model object. This object is then used consistently by Resolvers (for the API) and Services (for database interaction).

## Conclusion

You've reached the end of the tutorial! In this final chapter, you learned about the **Core Data Models**, the essential **blueprints** that define the structure of key concepts within the `gateway`.

*   Models like `User`, `Domain`, `Variable`, `Algorithm`, `Experiment`, and the `ResultUnion` types (`TableResult`, etc.) provide standard structures.
*   They ensure **consistency** across different parts of the application: how data is fetched by connectors, exposed via the GraphQL API, and stored in the database.
*   They act as the common language, making the system more predictable, maintainable, and easier to understand.

Throughout this tutorial, you've journeyed from the user-facing [GraphQL API Layer](01_graphql_api_layer_.md), through securing access with [Authentication & Authorization](02_authentication___authorization_.md), managing users ([User Service & Entity](03_user_service___entity_.md)) and computations ([Experiment Management](04_experiment_management_.md)), understanding how the gateway talks to backends ([Engine & Connectors](05_engine___connectors_.md)) and handles their results ([Result Handlers (Engine Connectors)](06_result_handlers__engine_connectors_.md)), finally arriving at the fundamental **Core Data Models** that underpin it all.

We hope this gives you a solid foundation for understanding and working with the `gateway` project!

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)