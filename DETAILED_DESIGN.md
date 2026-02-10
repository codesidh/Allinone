# CHC Insight CRM - Detailed Design Document

## Executive Summary

This document provides detailed technical specifications for the CHC Insight CRM system, including component designs, data models, API contracts, and implementation patterns.

**Version:** 1.1.0  
**Last Updated:** February 2026  
**Classification:** Internal Technical Documentation

---

## 1. Frontend Component Design

### 1.1 Component Hierarchy

```
App (layout.tsx)
├── Providers
│   ├── ThemeProvider
│   ├── QueryProvider (TanStack Query)
│   └── AuthProvider
├── Layout Components
│   ├── AppLayout
│   ├── SiteHeader
│   ├── AppSidebar
│   └── Breadcrumb
└── Page Components
    ├── Dashboard
    ├── FormManagement
    ├── Members
    ├── Providers
    ├── ProgressNotes
    ├── UMConsolidator
    └── Admin
```

### 1.2 Authentication Components

```typescript
// ProtectedRoute - Wraps all authenticated pages
interface ProtectedRouteProps {
  children: React.ReactNode;
}

// RoleGuard - Enforces role-based access
interface RoleGuardProps {
  children: React.ReactNode;
  requiredAccess: keyof RoleAccessConfig;
}

// Usage Pattern (MANDATORY for all protected pages)
export default function MyPage() {
  return (
    <ProtectedRoute>
      <RoleGuard requiredAccess="canAccessMyModule">
        <MyPageContent />
      </RoleGuard>
    </ProtectedRoute>
  );
}
```

### 1.3 Data Table Component System

```typescript
// Generic DataTable Component (components/ui/data-table/)
interface DataTableProps<TData, TValue> {
  // Required props
  columns: ColumnDef<TData, TValue>[];
  data: TData[];
  totalItems: number;
  currentPage: number;
  pageSize: number;
  onPageChange: (page: number) => void;
  
  // Optional configuration
  loading?: boolean;
  loadingMessage?: string;
  emptyMessage?: string;
  itemLabel?: string;
  onPageSizeChange?: (pageSize: number) => void;
  
  // Toolbar (module-specific)
  toolbar?: (props: { table: Table<TData> }) => React.ReactNode;
  
  // Row interactions
  onRowClick?: (row: TData) => void;
  enableRowSelection?: boolean;
}

// Module-specific implementation (3 files per module)
// 1. columns.tsx - Column definitions
// 2. data-table-toolbar.tsx - Module-specific filters
// 3. data-table.tsx - Thin wrapper (optional)
```

### 1.4 Form Components

```typescript
// React Hook Form + Zod Pattern
interface FormComponentProps<T extends z.ZodType> {
  schema: T;
  defaultValues: z.infer<T>;
  onSubmit: (data: z.infer<T>) => Promise<void>;
}

// Usage
const form = useForm<CreateUserData>({
  resolver: zodResolver(CreateUserSchema),
  defaultValues: {
    email: '',
    firstName: '',
    lastName: '',
    role: 'user',
  },
});
```

### 1.5 Custom Hooks Design

```typescript
// Server State Hooks (TanStack Query)
function useFormCategories(options?: UseFormHierarchyOptions) {
  return useQuery({
    queryKey: ['formCategories', options],
    queryFn: () => formHierarchyApi.getCategories(options),
    staleTime: 5 * 60 * 1000,
    gcTime: 10 * 60 * 1000,
  });
}

// Mutation Hooks with Optimistic Updates
function useUpdateFormTemplate() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: (data) => formHierarchyApi.updateTemplate(data.id, data),
    onMutate: async (newData) => {
      await queryClient.cancelQueries({ queryKey: ['formTemplates', newData.id] });
      const previous = queryClient.getQueryData(['formTemplates', newData.id]);
      queryClient.setQueryData(['formTemplates', newData.id], (old) => ({ ...old, ...newData }));
      return { previous };
    },
    onError: (err, newData, context) => {
      queryClient.setQueryData(['formTemplates', newData.id], context?.previous);
    },
    onSettled: (data, error, variables) => {
      queryClient.invalidateQueries({ queryKey: ['formTemplates', variables.id] });
    },
  });
}

// Local UI State Hooks
function useModalState() {
  const [isOpen, setIsOpen] = useState(false);
  const open = useCallback(() => setIsOpen(true), []);
  const close = useCallback(() => setIsOpen(false), []);
  return { isOpen, open, close };
}
```

---

## 2. Backend Service Design

### 2.1 Service Layer Architecture

```typescript
// Base Service Pattern
abstract class BaseService {
  protected db: UniversalDatabaseService;
  
  constructor() {
    this.db = UniversalDatabaseService.getInstance();
  }
  
  protected generateId(): string {
    return crypto.randomUUID();
  }
  
  protected handleDatabaseError(error: unknown): ApiResponse {
    console.error('Database error:', error);
    return {
      success: false,
      error: {
        code: 'DATABASE_ERROR',
        message: 'Database operation failed',
        details: error instanceof Error ? error.message : 'Unknown error'
      }
    };
  }
}
```

### 2.2 CRUD Service Implementation

```typescript
class FormHierarchyService extends BaseService {
  // READ - Single item
  async getCategoryById(id: string, tenantId: string): Promise<ApiResponse<FormCategory>> {
    try {
      const rows = await this.db.rawQuery(`
        SELECT * FROM form_categories 
        WHERE id = @id AND tenant_id = @tenantId AND is_active = 1
      `, { id, tenantId });
      
      if (rows.length === 0) {
        return { success: false, error: { code: 'NOT_FOUND', message: 'Category not found' } };
      }
      
      return { success: true, data: this.formatCategory(rows[0]) };
    } catch (error) {
      return this.handleDatabaseError(error);
    }
  }
  
  // READ - List with pagination
  async getCategories(tenantId: string, page: number, limit: number): Promise<ApiResponse<PaginatedResult>> {
    try {
      const offset = (page - 1) * limit;
      
      const countResult = await this.db.rawQuery(`
        SELECT COUNT(*) as total FROM form_categories 
        WHERE tenant_id = @tenantId AND is_active = 1
      `, { tenantId });
      
      const rows = await this.db.rawQuery(`
        SELECT * FROM form_categories 
        WHERE tenant_id = @tenantId AND is_active = 1
        ORDER BY name
        OFFSET @offset ROWS FETCH NEXT @limit ROWS ONLY
      `, { tenantId, offset, limit });
      
      return {
        success: true,
        data: rows.map(this.formatCategory),
        metadata: { total: countResult[0].total, page, limit }
      };
    } catch (error) {
      return this.handleDatabaseError(error);
    }
  }
  
  // CREATE
  async createCategory(data: CreateCategoryData, userId: string, tenantId: string): Promise<ApiResponse<FormCategory>> {
    try {
      const categoryData = {
        id: this.generateId(),
        tenant_id: tenantId,
        name: data.name,
        description: data.description,
        is_active: true,
        created_by: userId,
        updated_by: userId,
        created_at: new Date(),
        updated_at: new Date(),
      };
      
      const result = await this.db.insert('form_categories', categoryData);
      return { success: true, data: this.formatCategory(result.rows[0]) };
    } catch (error) {
      return this.handleDatabaseError(error);
    }
  }
  
  // UPDATE
  async updateCategory(id: string, data: UpdateCategoryData, userId: string, tenantId: string): Promise<ApiResponse<FormCategory>> {
    try {
      const result = await this.db.update(
        'form_categories',
        { ...data, updated_by: userId, updated_at: new Date() },
        { id, tenant_id: tenantId }
      );
      
      if (result.rowsAffected === 0) {
        return { success: false, error: { code: 'NOT_FOUND', message: 'Category not found' } };
      }
      
      return this.getCategoryById(id, tenantId);
    } catch (error) {
      return this.handleDatabaseError(error);
    }
  }
  
  // DELETE (soft delete)
  async deleteCategory(id: string, userId: string, tenantId: string): Promise<ApiResponse<void>> {
    try {
      const result = await this.db.update(
        'form_categories',
        { is_active: false, updated_by: userId, updated_at: new Date() },
        { id, tenant_id: tenantId }
      );
      
      if (result.rowsAffected === 0) {
        return { success: false, error: { code: 'NOT_FOUND', message: 'Category not found' } };
      }
      
      return { success: true };
    } catch (error) {
      return this.handleDatabaseError(error);
    }
  }
}
```

### 2.3 Controller Implementation Pattern

```typescript
class FormHierarchyController {
  constructor(private service: FormHierarchyService) {}
  
  getFormCategories = async (req: Request, res: Response): Promise<void> => {
    // 1. Validate authentication context
    const user = req.user as UserContext;
    if (!user?.tenantId) {
      res.status(401).json({
        success: false,
        error: { code: 'UNAUTHORIZED', message: 'Authentication required' }
      });
      return;
    }
    
    // 2. Extract and validate query parameters
    const { page = 1, limit = 10, isActive } = req.query;
    
    // 3. Call service layer
    const result = await this.service.getCategories(
      user.tenantId,
      Number(page),
      Number(limit)
    );
    
    // 4. Return response with appropriate status
    res.status(result.success ? 200 : 500).json(result);
  }
  
  createFormCategory = async (req: Request, res: Response): Promise<void> => {
    // 1. Validate authentication context
    const user = req.user as UserContext;
    if (!user?.tenantId || !user?.userId) {
      res.status(401).json({
        success: false,
        error: { code: 'UNAUTHORIZED', message: 'Authentication required' }
      });
      return;
    }
    
    // 2. Validate request body with Zod schema
    const validationResult = CreateCategorySchema.safeParse(req.body);
    if (!validationResult.success) {
      res.status(400).json({
        success: false,
        error: {
          code: 'VALIDATION_ERROR',
          message: 'Invalid request data',
          details: validationResult.error.format()
        }
      });
      return;
    }
    
    // 3. Call service layer
    const result = await this.service.createCategory(
      validationResult.data,
      user.userId,
      user.tenantId
    );
    
    // 4. Map error codes to HTTP status
    const statusCode = result.success ? 201 : this.mapErrorToStatus(result.error?.code);
    res.status(statusCode).json(result);
  }
  
  private mapErrorToStatus(errorCode?: string): number {
    const statusMap: Record<string, number> = {
      'NOT_FOUND': 404,
      'ALREADY_EXISTS': 409,
      'VALIDATION_ERROR': 400,
      'UNAUTHORIZED': 401,
      'FORBIDDEN': 403,
      'DATABASE_ERROR': 500,
    };
    return statusMap[errorCode || ''] || 500;
  }
}
```

### 2.4 Validation Schema Design

```typescript
// shared/validation/base.schemas.ts
export const BaseEntitySchema = z.object({
  id: z.string().uuid(),
  tenantId: z.string().uuid(),
});

export const AuditTrailSchema = z.object({
  createdBy: z.string().uuid(),
  updatedBy: z.string().uuid(),
  createdAt: z.date(),
  updatedAt: z.date(),
});

// modules/form-hierarchy/validation/form-hierarchy.schemas.ts
const FormCategorySchema = BaseEntitySchema.merge(AuditTrailSchema).extend({
  name: z.string().min(1).max(255),
  description: z.string().max(500).optional(),
  isActive: z.boolean().default(true),
});

// Create schema - excludes server-managed fields
const CreateCategorySchema = FormCategorySchema.omit({
  id: true,
  tenantId: true,
  createdBy: true,
  updatedBy: true,
  createdAt: true,
  updatedAt: true,
});

// Update schema - partial of create schema
const UpdateCategorySchema = CreateCategorySchema.partial();

export type FormCategory = z.infer<typeof FormCategorySchema>;
export type CreateCategoryData = z.infer<typeof CreateCategorySchema>;
export type UpdateCategoryData = z.infer<typeof UpdateCategorySchema>;
```

---

## 3. Database Schema Design

### 3.1 Core Tables

#### tenants
```sql
CREATE TABLE tenants (
  id uniqueidentifier PRIMARY KEY DEFAULT newid(),
  name nvarchar(255) NOT NULL,
  subdomain nvarchar(100) UNIQUE,
  configuration nvarchar(max) DEFAULT '{}',  -- JSON: theme, features, integrations
  is_active bit DEFAULT 1,
  created_at datetime2 DEFAULT CURRENT_TIMESTAMP,
  updated_at datetime2 DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_tenants_is_active ON tenants (is_active);
CREATE INDEX idx_tenants_subdomain ON tenants (subdomain);
```

#### users
```sql
CREATE TABLE users (
  id uniqueidentifier PRIMARY KEY DEFAULT newid(),
  tenant_id uniqueidentifier NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  email nvarchar(255) NOT NULL,
  password_hash nvarchar(255) NOT NULL,
  first_name nvarchar(100) NOT NULL,
  last_name nvarchar(100) NOT NULL,
  is_active bit DEFAULT 1,
  last_login datetime2,
  region nvarchar(100),
  member_panel nvarchar(max),      -- JSON array
  provider_network nvarchar(max),  -- JSON array
  created_by uniqueidentifier REFERENCES users(id),
  updated_by uniqueidentifier REFERENCES users(id),
  created_at datetime2 DEFAULT CURRENT_TIMESTAMP,
  updated_at datetime2 DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT uq_users_tenant_email UNIQUE (tenant_id, email)
);

-- Indexes
CREATE INDEX idx_users_tenant_id ON users (tenant_id);
CREATE INDEX idx_users_email ON users (email);
CREATE INDEX idx_users_is_active ON users (is_active);
CREATE INDEX idx_users_region ON users (region);
```

#### roles
```sql
CREATE TABLE roles (
  id uniqueidentifier PRIMARY KEY DEFAULT newid(),
  tenant_id uniqueidentifier NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  name nvarchar(100) NOT NULL,
  description nvarchar(500),
  permissions nvarchar(max) NOT NULL DEFAULT '[]',  -- JSON array of permissions
  created_at datetime2 DEFAULT CURRENT_TIMESTAMP,
  updated_at datetime2 DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT uq_roles_tenant_name UNIQUE (tenant_id, name)
);
```

### 3.2 Form Hierarchy Tables

#### form_categories (Level 1)
```sql
CREATE TABLE form_categories (
  id uniqueidentifier PRIMARY KEY DEFAULT newid(),
  tenant_id uniqueidentifier NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  name nvarchar(255) NOT NULL,           -- 'Cases', 'Assessments'
  description nvarchar(500),
  is_active bit DEFAULT 1,
  created_by uniqueidentifier REFERENCES users(id),
  updated_by uniqueidentifier REFERENCES users(id),
  created_at datetime2 DEFAULT CURRENT_TIMESTAMP,
  updated_at datetime2 DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT uq_form_categories_tenant_name UNIQUE (tenant_id, name)
);

-- Indexes
CREATE INDEX idx_form_categories_tenant_id ON form_categories (tenant_id);
CREATE INDEX idx_form_categories_is_active ON form_categories (is_active);
```

#### form_types (Level 2)
```sql
CREATE TABLE form_types (
  id uniqueidentifier PRIMARY KEY DEFAULT newid(),
  tenant_id uniqueidentifier NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  category_id uniqueidentifier NOT NULL REFERENCES form_categories(id),
  name nvarchar(255) NOT NULL,           -- 'BH Referrals', 'Health Risk (HDM)'
  description nvarchar(500),
  business_rules nvarchar(max) DEFAULT '[]',  -- JSON: conditional logic, validations
  is_active bit DEFAULT 1,
  created_by uniqueidentifier REFERENCES users(id),
  updated_by uniqueidentifier REFERENCES users(id),
  created_at datetime2 DEFAULT CURRENT_TIMESTAMP,
  updated_at datetime2 DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT uq_form_types_category_name UNIQUE (category_id, name)
);

-- Indexes
CREATE INDEX idx_form_types_tenant_id ON form_types (tenant_id);
CREATE INDEX idx_form_types_category_id ON form_types (category_id);
CREATE INDEX idx_form_types_is_active ON form_types (is_active);
```

#### form_templates (Level 3)
```sql
CREATE TABLE form_templates (
  id uniqueidentifier PRIMARY KEY DEFAULT newid(),
  tenant_id uniqueidentifier NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  type_id uniqueidentifier NOT NULL REFERENCES form_types(id),
  name nvarchar(255) NOT NULL,           -- 'BH Initial Referral v1.0'
  description nvarchar(1000),
  version int DEFAULT 1,
  template_data nvarchar(max) DEFAULT '{}',   -- JSON: questions, sections, layout
  workflow_config nvarchar(max) DEFAULT '{}', -- JSON: approval steps, assignments
  is_active bit DEFAULT 1,
  effective_date datetime2 NOT NULL,
  expiration_date datetime2,
  created_by uniqueidentifier REFERENCES users(id),
  updated_by uniqueidentifier REFERENCES users(id),
  created_at datetime2 DEFAULT CURRENT_TIMESTAMP,
  updated_at datetime2 DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT uq_form_templates_type_name_version UNIQUE (type_id, name, version)
);

-- Indexes
CREATE INDEX idx_form_templates_tenant_id ON form_templates (tenant_id);
CREATE INDEX idx_form_templates_type_id ON form_templates (type_id);
CREATE INDEX idx_form_templates_is_active ON form_templates (is_active);
CREATE INDEX idx_form_templates_effective_date ON form_templates (effective_date);
CREATE INDEX idx_form_templates_version ON form_templates (version);
```

#### form_instances (Level 4)
```sql
CREATE TABLE form_instances (
  id uniqueidentifier PRIMARY KEY DEFAULT newid(),
  tenant_id uniqueidentifier NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  template_id uniqueidentifier NOT NULL REFERENCES form_templates(id),
  member_id nvarchar(50),                -- Reference to member_staging
  provider_id nvarchar(50),              -- Reference to provider_staging
  assigned_to uniqueidentifier REFERENCES users(id),
  status nvarchar(50) DEFAULT 'draft',   -- draft, in_progress, submitted, approved, rejected
  response_data nvarchar(max) DEFAULT '[]',   -- JSON: answers to questions
  context_data nvarchar(max) DEFAULT '{}',    -- JSON: additional context
  due_date datetime2,
  submitted_at datetime2,
  approved_at datetime2,
  rejected_at datetime2,
  rejection_reason nvarchar(1000),
  created_by uniqueidentifier REFERENCES users(id),
  updated_by uniqueidentifier REFERENCES users(id),
  created_at datetime2 DEFAULT CURRENT_TIMESTAMP,
  updated_at datetime2 DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT chk_form_instances_status CHECK (
    status IN ('draft', 'in_progress', 'submitted', 'approved', 'rejected', 'cancelled')
  )
);

-- Indexes
CREATE INDEX idx_form_instances_tenant_id ON form_instances (tenant_id);
CREATE INDEX idx_form_instances_template_id ON form_instances (template_id);
CREATE INDEX idx_form_instances_assigned_to ON form_instances (assigned_to);
CREATE INDEX idx_form_instances_status ON form_instances (status);
CREATE INDEX idx_form_instances_member_id ON form_instances (member_id);
CREATE INDEX idx_form_instances_provider_id ON form_instances (provider_id);
CREATE INDEX idx_form_instances_due_date ON form_instances (due_date);
```

### 3.3 Member & Provider Tables

#### member_staging
```sql
CREATE TABLE member_staging (
  member_id nvarchar(50) PRIMARY KEY,
  tenant_id uniqueidentifier NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  medicaid_id nvarchar(50) NOT NULL,
  hcin_id nvarchar(50) NOT NULL,
  first_name nvarchar(100) NOT NULL,
  last_name nvarchar(100) NOT NULL,
  date_of_birth datetime2 NOT NULL,
  sex nvarchar(1) CHECK (sex IN ('M', 'F')),
  plan_id nvarchar(50) NOT NULL,
  plan_category nvarchar(50) CHECK (plan_category IN ('Medical', 'RX', 'Vision')),
  plan_type nvarchar(50) CHECK (plan_type IN ('NFCE', 'NFI')),
  plan_sub_type nvarchar(50) CHECK (plan_sub_type IN ('HCBS', 'NF', 'NFI')),
  waiver_code nvarchar(10) CHECK (waiver_code IN ('20', '37', '38', '39')),
  waiver_effective_date datetime2,
  waiver_term_date datetime2,
  member_zone nvarchar(10) CHECK (member_zone IN ('SW', 'SE', 'NE', 'NW', 'LC')),
  pics_score int CHECK (pics_score >= 0 AND pics_score <= 100),
  assigned_coordinator nvarchar(50),
  contact_info nvarchar(max) DEFAULT '{}',  -- JSON: phone, address, email
  last_updated datetime2 DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_member_staging_tenant_id ON member_staging (tenant_id);
CREATE INDEX idx_member_staging_medicaid_id ON member_staging (medicaid_id);
CREATE INDEX idx_member_staging_hcin_id ON member_staging (hcin_id);
CREATE INDEX idx_member_staging_first_name ON member_staging (first_name);
CREATE INDEX idx_member_staging_last_name ON member_staging (last_name);
CREATE INDEX idx_member_staging_assigned_coordinator ON member_staging (assigned_coordinator);
```

#### provider_staging
```sql
CREATE TABLE provider_staging (
  provider_id nvarchar(50) PRIMARY KEY,
  tenant_id uniqueidentifier NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  name nvarchar(255) NOT NULL,
  npi nvarchar(10) NOT NULL CHECK (LEN(npi) = 10),
  taxonomy nvarchar(50),
  provider_entity nvarchar(255),
  provider_type nvarchar(255),
  provider_type_code nvarchar(50),
  organization_type nvarchar(255),
  specialty nvarchar(255),
  specialty_code nvarchar(50),
  sub_specialty nvarchar(255),
  network_status nvarchar(50) DEFAULT 'in_network' 
    CHECK (network_status IN ('in_network', 'out_of_network', 'pending', 'terminated')),
  contact_info nvarchar(max) DEFAULT '{}',  -- JSON: phone, address, fax
  relationship_specialist_name nvarchar(255),
  last_updated datetime2 DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_provider_staging_tenant_id ON provider_staging (tenant_id);
CREATE INDEX idx_provider_staging_name ON provider_staging (name);
CREATE INDEX idx_provider_staging_npi ON provider_staging (npi);
CREATE INDEX idx_provider_staging_specialty ON provider_staging (specialty);
CREATE INDEX idx_provider_staging_network_status ON provider_staging (network_status);
```

### 3.4 UM Consolidator Tables

#### um_reviews
```sql
CREATE TABLE um_reviews (
  id uniqueidentifier PRIMARY KEY DEFAULT newid(),
  tenant_id uniqueidentifier NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  
  -- Participant Information
  participant_id nvarchar(50) NOT NULL,
  participant_first_name nvarchar(100),
  participant_last_name nvarchar(100),
  participant_zone nvarchar(50),
  
  -- Review Assignment
  reviewer_id uniqueidentifier REFERENCES users(id),
  procedure_code nvarchar(50) NOT NULL,
  
  -- Review Details
  current_care_plan_hours decimal(10,2) NOT NULL,
  request_type nvarchar(20) NOT NULL CHECK (request_type IN ('Increase', 'Decrease', 'Maintain')),
  requested_hours_per_week decimal(10,2) NOT NULL,
  approved_hours_per_week decimal(10,2),
  decision nvarchar(50) CHECK (decision IN (
    'Approval', 'Denial', 'Denial - With Reduction', 
    'Denial - Would have Reduced', 'Flagged reduction', 
    'More information', 'Reduction'
  )),
  reduction_hours decimal(10,2),
  assessment_type nvarchar(50) CHECK (assessment_type IN ('F2F Assessment', 'Telephonic Assessment')),
  psst_hours decimal(10,2),
  
  -- Calculated Fields
  requires_secondary_review bit NOT NULL DEFAULT 0,
  hours_difference_current decimal(10,2),
  hours_difference_requested decimal(10,2),
  
  -- Secondary Review
  secondary_review_decision nvarchar(50),
  updated_ltss_hours decimal(10,2),
  updated_difference decimal(10,2),
  
  -- Status and Workflow
  status nvarchar(50) NOT NULL DEFAULT 'Assigned' 
    CHECK (status IN ('Assigned', 'In Progress', 'Secondary Review', 'Review Completed')),
  
  -- Notes
  notes nvarchar(max),
  secondary_review_notes nvarchar(max),       -- Notes from secondary review
  
  -- Audit Fields
  review_date datetime2 DEFAULT GETUTCDATE(),
  week_of_year int,
  created_by uniqueidentifier NOT NULL REFERENCES users(id),
  updated_by uniqueidentifier REFERENCES users(id),
  created_at datetime2 DEFAULT GETUTCDATE(),
  updated_at datetime2 DEFAULT GETUTCDATE()
);

-- Performance Indexes
CREATE INDEX IX_um_reviews_tenant_id ON um_reviews (tenant_id);
CREATE INDEX IX_um_reviews_reviewer_id ON um_reviews (reviewer_id);
CREATE INDEX IX_um_reviews_status ON um_reviews (status);
CREATE INDEX IX_um_reviews_participant_id ON um_reviews (participant_id);
CREATE INDEX IX_um_reviews_procedure_code ON um_reviews (procedure_code);
CREATE INDEX IX_um_reviews_created_at ON um_reviews (created_at DESC);
CREATE INDEX IX_um_reviews_review_date ON um_reviews (review_date DESC);
CREATE INDEX IX_um_reviews_tenant_status ON um_reviews (tenant_id, status);
CREATE INDEX IX_um_reviews_tenant_reviewer ON um_reviews (tenant_id, reviewer_id);
CREATE INDEX IX_um_reviews_reviewer_status ON um_reviews (reviewer_id, status);
CREATE INDEX IX_um_reviews_secondary_review ON um_reviews (requires_secondary_review, status);
CREATE INDEX IX_um_reviews_tenant_created ON um_reviews (tenant_id, created_at DESC);
```

#### appeals
```sql
CREATE TABLE appeals (
  id uniqueidentifier PRIMARY KEY DEFAULT newid(),
  tenant_id uniqueidentifier NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  um_review_id uniqueidentifier NOT NULL REFERENCES um_reviews(id),
  
  -- Participant Information
  participant_id nvarchar(50) NOT NULL,
  
  -- Appeal Assignment
  appeal_reviewer_id uniqueidentifier REFERENCES users(id),
  
  -- Appeal Details
  appeal_hearing datetime2 NOT NULL,
  denied_hours decimal(10,2) NOT NULL,
  appeal_type nvarchar(50) NOT NULL CHECK (appeal_type IN (
    'Deny Increase/Reduction', 'DME', 'Home Adaptation', 
    'Increase Request', 'Initial Request', 'Reduction'
  )),
  appeal_results nvarchar(50) CHECK (appeal_results IN (
    'Overturned', 'Partially Overturned', 'Removed From Schedule', 
    'Upheld', 'Withdrawn'
  )),
  hours_after_grievance decimal(10,2),
  reason_for_pas_hours nvarchar(max),
  
  -- Hearing Participants
  employee_voter_id uniqueidentifier REFERENCES users(id),
  medical_director_id uniqueidentifier REFERENCES users(id),
  um_reviewer_id uniqueidentifier REFERENCES users(id),
  remediation_effective_date datetime2,
  
  -- Status
  status nvarchar(50) NOT NULL DEFAULT 'Submitted' 
    CHECK (status IN ('Submitted', 'Hearing', 'Decision')),
  
  -- Audit Fields
  year_of_review int,
  created_by uniqueidentifier NOT NULL REFERENCES users(id),
  updated_by uniqueidentifier REFERENCES users(id),
  created_at datetime2 DEFAULT GETUTCDATE(),
  updated_at datetime2 DEFAULT GETUTCDATE()
);

-- Performance Indexes
CREATE INDEX IX_appeals_tenant_id ON appeals (tenant_id);
CREATE INDEX IX_appeals_um_review_id ON appeals (um_review_id);
CREATE INDEX IX_appeals_reviewer_id ON appeals (appeal_reviewer_id);
CREATE INDEX IX_appeals_status ON appeals (status);
CREATE INDEX IX_appeals_participant_id ON appeals (participant_id);
CREATE INDEX IX_appeals_appeal_type ON appeals (appeal_type);
CREATE INDEX IX_appeals_appeal_hearing ON appeals (appeal_hearing DESC);
CREATE INDEX IX_appeals_created_at ON appeals (created_at DESC);
CREATE INDEX IX_appeals_tenant_status ON appeals (tenant_id, status);
CREATE INDEX IX_appeals_tenant_reviewer ON appeals (tenant_id, appeal_reviewer_id);
CREATE INDEX IX_appeals_reviewer_status ON appeals (appeal_reviewer_id, status);
CREATE INDEX IX_appeals_year_review ON appeals (year_of_review);
```

#### vw_um_reviews_with_participant (VIEW)
```sql
-- Optimized view for efficient querying with participant details
CREATE VIEW vw_um_reviews_with_participant AS
SELECT 
  ur.*,
  ms.first_name as participant_first_name,
  ms.last_name as participant_last_name,
  ms.member_zone as participant_zone,
  ms.medicaid_id,
  ms.hcin_id,
  u.first_name as reviewer_first_name,
  u.last_name as reviewer_last_name
FROM um_reviews ur
LEFT JOIN member_staging ms ON ur.participant_id IN (ms.member_id, ms.medicaid_id, ms.hcin_id)
LEFT JOIN users u ON ur.reviewer_id = u.id;
```

#### vw_appeals_with_details (VIEW)
```sql
-- Optimized view for appeals with related details
CREATE VIEW vw_appeals_with_details AS
SELECT 
  a.*,
  ur.current_care_plan_hours,
  ur.requested_hours_per_week,
  ur.approved_hours_per_week,
  ur.decision as um_decision,
  ms.first_name as member_first_name,
  ms.last_name as member_last_name,
  ms.member_zone as member_zone,
  ar.first_name + ' ' + ar.last_name as appeal_reviewer_name,
  ev.first_name + ' ' + ev.last_name as employee_voter_name,
  md.first_name + ' ' + md.last_name as medical_director_name,
  umr.first_name + ' ' + umr.last_name as um_reviewer_name
FROM appeals a
INNER JOIN um_reviews ur ON a.um_review_id = ur.id
LEFT JOIN member_staging ms ON a.participant_id IN (ms.member_id, ms.medicaid_id, ms.hcin_id)
LEFT JOIN users ar ON a.appeal_reviewer_id = ar.id
LEFT JOIN users ev ON a.employee_voter_id = ev.id
LEFT JOIN users md ON a.medical_director_id = md.id
LEFT JOIN users umr ON a.um_reviewer_id = umr.id;
```

#### um_audit
```sql
CREATE TABLE um_audit (
  id uniqueidentifier PRIMARY KEY DEFAULT newid(),
  um_review_id uniqueidentifier NOT NULL REFERENCES um_reviews(id),
  user_id uniqueidentifier NOT NULL REFERENCES users(id),
  action nvarchar(50) NOT NULL,           -- 'CREATE', 'UPDATE', 'STATUS_CHANGE', 'ASSIGN'
  field_changed nvarchar(100),
  old_value nvarchar(max),
  new_value nvarchar(max),
  created_at datetime2 DEFAULT GETUTCDATE()
);

-- Performance Indexes
CREATE INDEX IX_um_audit_um_review_id ON um_audit (um_review_id);
CREATE INDEX IX_um_audit_user_id ON um_audit (user_id);
CREATE INDEX IX_um_audit_created_at ON um_audit (created_at DESC);
```

### 3.5 Progress Notes Tables

#### progress_notes_templates
```sql
CREATE TABLE progress_notes_templates (
  id uniqueidentifier PRIMARY KEY DEFAULT newid(),
  tenant_id uniqueidentifier NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  name nvarchar(255) NOT NULL,
  description nvarchar(1000),
  decision_category nvarchar(100) NOT NULL,  -- One of 7 template types
  note_template nvarchar(max) NOT NULL,      -- Template with {{placeholders}}
  questions nvarchar(max) DEFAULT '[]',      -- JSON: additional questions
  predefined_text_config nvarchar(max) DEFAULT '{}',  -- JSON: predefined text options
  version int DEFAULT 1,
  is_active bit DEFAULT 1,
  created_by uniqueidentifier NOT NULL REFERENCES users(id),
  updated_by uniqueidentifier REFERENCES users(id),
  created_at datetime2 DEFAULT GETUTCDATE(),
  updated_at datetime2 DEFAULT GETUTCDATE(),
  
  CONSTRAINT chk_decision_category CHECK (decision_category IN (
    'Request for Additional Information',
    'Approval to continue current plan of care',
    'Approval of increase',
    'Complete Denial',
    'Partially Deny an Increase',
    'Reduction of Current plan of care',
    'Denial of Increase & Reduction of current plan of care'
  ))
);

-- Performance Indexes
CREATE INDEX IX_progress_notes_templates_tenant_id ON progress_notes_templates (tenant_id);
CREATE INDEX IX_progress_notes_templates_decision_category ON progress_notes_templates (decision_category);
CREATE INDEX IX_progress_notes_templates_is_active ON progress_notes_templates (is_active);
CREATE INDEX IX_progress_notes_templates_tenant_active ON progress_notes_templates (tenant_id, is_active);
CREATE INDEX IX_progress_notes_templates_version ON progress_notes_templates (tenant_id, version DESC);
```

#### progress_notes_instances
```sql
CREATE TABLE progress_notes_instances (
  id uniqueidentifier PRIMARY KEY DEFAULT newid(),
  tenant_id uniqueidentifier NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  template_id uniqueidentifier REFERENCES progress_notes_templates(id),  -- Optional, auto-selected if not provided
  reviewer_id uniqueidentifier NOT NULL REFERENCES users(id),
  member_id nvarchar(50),                    -- Reference to member_staging
  provider_id nvarchar(50),                  -- Reference to provider_staging
  case_id uniqueidentifier,                  -- Reference to form_instances
  care_plan_review_type nvarchar(50),        -- 'increase', 'decrease', 'continue'
  question_responses nvarchar(max) DEFAULT '{}',   -- JSON: answers to predefined questions
  generated_notes nvarchar(max),             -- JSON: generated note content per template
  status nvarchar(50) DEFAULT 'draft' CHECK (status IN ('draft', 'submitted')),
  
  -- Time Tracking
  submitted_at datetime2,                    -- When notes were submitted
  started_at datetime2,                      -- When reviewer opened the form
  completed_at datetime2,                    -- When notes were generated/submitted
  time_spent_minutes int,                    -- Calculated duration
  
  -- Audit Fields
  created_by uniqueidentifier NOT NULL REFERENCES users(id),
  updated_by uniqueidentifier REFERENCES users(id),
  created_at datetime2 DEFAULT GETUTCDATE(),
  updated_at datetime2 DEFAULT GETUTCDATE()
);

-- Performance Indexes
CREATE INDEX IX_progress_notes_instances_tenant_id ON progress_notes_instances (tenant_id);
CREATE INDEX IX_progress_notes_instances_template_id ON progress_notes_instances (template_id);
CREATE INDEX IX_progress_notes_instances_reviewer_id ON progress_notes_instances (reviewer_id);
CREATE INDEX IX_progress_notes_instances_status ON progress_notes_instances (status);
CREATE INDEX IX_progress_notes_instances_member_id ON progress_notes_instances (member_id);
CREATE INDEX IX_progress_notes_instances_case_id ON progress_notes_instances (case_id);
CREATE INDEX IX_progress_notes_instances_created_at ON progress_notes_instances (created_at DESC);
CREATE INDEX IX_progress_notes_instances_started_at ON progress_notes_instances (started_at);
CREATE INDEX IX_progress_notes_instances_completed_at ON progress_notes_instances (completed_at);
CREATE INDEX IX_progress_notes_instances_time_spent ON progress_notes_instances (time_spent_minutes);
CREATE INDEX IX_progress_notes_instances_reviewer_time ON progress_notes_instances (reviewer_id, completed_at DESC, time_spent_minutes);
```

#### progress_notes_question_templates
```sql
CREATE TABLE progress_notes_question_templates (
  id uniqueidentifier PRIMARY KEY DEFAULT newid(),
  tenant_id uniqueidentifier NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  version int NOT NULL DEFAULT 1,
  questions_config nvarchar(max) NOT NULL,   -- JSON: predefined questions configuration
  is_active bit DEFAULT 1,
  created_by uniqueidentifier NOT NULL REFERENCES users(id),
  created_at datetime2 DEFAULT GETUTCDATE(),
  updated_at datetime2 DEFAULT GETUTCDATE()
);

-- Performance Indexes
CREATE INDEX IX_progress_notes_question_templates_tenant_id ON progress_notes_question_templates (tenant_id);
CREATE INDEX IX_progress_notes_question_templates_is_active ON progress_notes_question_templates (is_active);
CREATE INDEX IX_progress_notes_question_templates_tenant_active ON progress_notes_question_templates (tenant_id, is_active);
CREATE INDEX IX_progress_notes_question_templates_version ON progress_notes_question_templates (tenant_id, version DESC);
```

#### progress_notes_assignment_history
```sql
CREATE TABLE progress_notes_assignment_history (
  id uniqueidentifier PRIMARY KEY DEFAULT newid(),
  progress_note_id uniqueidentifier NOT NULL REFERENCES progress_notes_instances(id),
  from_reviewer_id uniqueidentifier REFERENCES users(id),
  to_reviewer_id uniqueidentifier NOT NULL REFERENCES users(id),
  assigned_by uniqueidentifier NOT NULL REFERENCES users(id),
  reason nvarchar(500),
  created_at datetime2 DEFAULT GETUTCDATE()
);

-- Performance Indexes
CREATE INDEX IX_progress_notes_assignment_history_note_id ON progress_notes_assignment_history (progress_note_id);
CREATE INDEX IX_progress_notes_assignment_history_from_reviewer ON progress_notes_assignment_history (from_reviewer_id);
CREATE INDEX IX_progress_notes_assignment_history_to_reviewer ON progress_notes_assignment_history (to_reviewer_id);
CREATE INDEX IX_progress_notes_assignment_history_created_at ON progress_notes_assignment_history (created_at DESC);
```

#### progress_notes_analytics
```sql
CREATE TABLE progress_notes_analytics (
  id uniqueidentifier PRIMARY KEY DEFAULT newid(),
  tenant_id uniqueidentifier NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  reviewer_id uniqueidentifier NOT NULL REFERENCES users(id),
  instance_id uniqueidentifier NOT NULL,      -- Reference to progress_notes_instances
  template_types_generated nvarchar(max),     -- JSON: template types used
  completion_time_minutes int,                -- Time to complete the note
  questions_answered int,                     -- Number of questions answered
  notes_copied_count int DEFAULT 0,           -- Number of times notes were copied
  care_plan_review_type nvarchar(50),         -- 'increase', 'decrease', 'continue'
  created_at datetime2 DEFAULT GETUTCDATE()
);

-- Performance Indexes
CREATE INDEX IX_progress_notes_analytics_tenant_id ON progress_notes_analytics (tenant_id);
CREATE INDEX IX_progress_notes_analytics_reviewer_id ON progress_notes_analytics (reviewer_id);
CREATE INDEX IX_progress_notes_analytics_instance_id ON progress_notes_analytics (instance_id);
CREATE INDEX IX_progress_notes_analytics_created_at ON progress_notes_analytics (created_at DESC);
CREATE INDEX IX_progress_notes_analytics_tenant_reviewer ON progress_notes_analytics (tenant_id, reviewer_id);
CREATE INDEX IX_progress_notes_analytics_tenant_created ON progress_notes_analytics (tenant_id, created_at DESC);
CREATE INDEX IX_progress_notes_analytics_reviewer_created ON progress_notes_analytics (reviewer_id, created_at DESC);
CREATE INDEX IX_progress_notes_analytics_dashboard ON progress_notes_analytics (tenant_id, reviewer_id, created_at DESC);
CREATE INDEX IX_progress_notes_analytics_completion_time ON progress_notes_analytics (completion_time_minutes);
CREATE INDEX IX_progress_notes_analytics_questions_answered ON progress_notes_analytics (questions_answered);
```

#### progress_notes_dashboard_metrics (VIEW)
```sql
-- Aggregated analytics view for dashboard performance
CREATE VIEW progress_notes_dashboard_metrics AS
SELECT 
  pni.tenant_id,
  pni.reviewer_id,
  COUNT(*) as total_notes,
  SUM(CASE WHEN pni.status = 'draft' THEN 1 ELSE 0 END) as draft_notes,
  SUM(CASE WHEN pni.status = 'completed' THEN 1 ELSE 0 END) as completed_notes,
  SUM(CASE WHEN pni.status = 'submitted' THEN 1 ELSE 0 END) as submitted_notes,
  SUM(CASE WHEN pni.created_at >= DATEADD(month, -1, GETUTCDATE()) THEN 1 ELSE 0 END) as this_month_notes,
  SUM(CASE WHEN pni.created_at >= DATEADD(week, -1, GETUTCDATE()) THEN 1 ELSE 0 END) as this_week_notes,
  AVG(CAST(pna.completion_time_minutes as float)) as avg_completion_time,
  AVG(CAST(pna.questions_answered as float)) as avg_questions_answered,
  SUM(pna.notes_copied_count) as total_notes_copied
FROM progress_notes_instances pni
LEFT JOIN progress_notes_analytics pna ON pni.id = pna.instance_id
GROUP BY pni.tenant_id, pni.reviewer_id;
```

#### progress_notes_monthly_summary
```sql
CREATE TABLE progress_notes_monthly_summary (
  id uniqueidentifier PRIMARY KEY DEFAULT newid(),
  tenant_id uniqueidentifier NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  reviewer_id uniqueidentifier NOT NULL REFERENCES users(id),
  year int NOT NULL,
  month int NOT NULL,
  total_notes int DEFAULT 0,
  completed_notes int DEFAULT 0,
  avg_completion_time_minutes decimal(10,2),
  total_template_types_used int DEFAULT 0,
  most_used_template_type nvarchar(100),
  created_at datetime2 DEFAULT GETUTCDATE(),
  
  CONSTRAINT UQ_progress_notes_monthly_summary UNIQUE (tenant_id, reviewer_id, year, month)
);

-- Performance Indexes
CREATE INDEX IX_progress_notes_monthly_summary_tenant_year_month ON progress_notes_monthly_summary (tenant_id, year DESC, month DESC);
CREATE INDEX IX_progress_notes_monthly_summary_reviewer_year_month ON progress_notes_monthly_summary (reviewer_id, year DESC, month DESC);
CREATE INDEX IX_progress_notes_monthly_summary_created_at ON progress_notes_monthly_summary (created_at DESC);
```

---

## 4. API Contract Specifications

### 4.1 Authentication API

#### POST /api/auth/login
```typescript
// Request
interface LoginRequest {
  email: string;      // Required, valid email format
  password: string;   // Required, min 8 characters
}

// Response (200 OK)
interface LoginResponse {
  success: true;
  data: {
    user: {
      id: string;
      email: string;
      firstName: string;
      lastName: string;
      roles: Role[];
      tenantId: string;
      isActive: boolean;
    };
    accessToken: string;   // JWT, expires in 1 hour
    refreshToken: string;  // JWT, expires in 7 days
    tokenType: 'Bearer';
  };
}

// Error Response (401 Unauthorized)
interface LoginErrorResponse {
  success: false;
  error: {
    code: 'INVALID_CREDENTIALS';
    message: 'Invalid email or password';
  };
}
```

#### POST /api/auth/refresh
```typescript
// Request
interface RefreshRequest {
  refreshToken: string;
}

// Response (200 OK)
interface RefreshResponse {
  success: true;
  data: {
    accessToken: string;
    refreshToken: string;
    expiresIn: number;     // Seconds until expiration
    tokenType: 'Bearer';
  };
}
```

#### GET /api/auth/me
```typescript
// Headers: Authorization: Bearer <accessToken>

// Response (200 OK)
interface CurrentUserResponse {
  success: true;
  data: {
    user: User;
  };
}
```

### 4.2 Form Hierarchy API

#### GET /api/form-categories
```typescript
// Query Parameters
interface GetCategoriesQuery {
  page?: number;      // Default: 1
  limit?: number;     // Default: 10, max: 100
  isActive?: boolean; // Filter by active status
}

// Response (200 OK)
interface GetCategoriesResponse {
  success: true;
  data: FormCategory[];
  metadata: {
    total: number;
    page: number;
    limit: number;
    timestamp: string;
  };
}
```

#### POST /api/form-categories
```typescript
// Request
interface CreateCategoryRequest {
  name: string;         // Required, 1-255 chars
  description?: string; // Optional, max 500 chars
}

// Response (201 Created)
interface CreateCategoryResponse {
  success: true;
  data: FormCategory;
}

// Error Response (409 Conflict)
interface DuplicateCategoryResponse {
  success: false;
  error: {
    code: 'ALREADY_EXISTS';
    message: 'Category with this name already exists';
  };
}
```

#### GET /api/form-categories/:categoryId/types
```typescript
// Response (200 OK)
interface GetTypesResponse {
  success: true;
  data: FormType[];
  metadata: {
    total: number;
    page: number;
    limit: number;
  };
}
```

#### POST /api/form-types/:typeId/templates
```typescript
// Request
interface CreateTemplateRequest {
  name: string;
  description?: string;
  templateData: {
    sections: Section[];
    questions: Question[];
  };
  workflowConfig?: {
    approvalSteps: ApprovalStep[];
    autoAssignment?: AssignmentRule[];
  };
  effectiveDate: string;  // ISO 8601 date
  expirationDate?: string;
}

// Response (201 Created)
interface CreateTemplateResponse {
  success: true;
  data: FormTemplate;
}
```

### 4.3 Member API

#### GET /api/members/search
```typescript
// Query Parameters
interface MemberSearchQuery {
  query?: string;           // Search term (name, ID)
  page?: number;
  limit?: number;
  zone?: string;            // Filter by zone
  planCategory?: string;    // Filter by plan category
  assignedCoordinator?: string;
}

// Response (200 OK)
interface MemberSearchResponse {
  success: true;
  data: Member[];
  metadata: {
    total: number;
    page: number;
    limit: number;
  };
}
```

#### POST /api/members/upload
```typescript
// Request: multipart/form-data
// Field: file (Excel file)

// Response (200 OK)
interface MemberUploadResponse {
  success: true;
  data: {
    imported: number;
    updated: number;
    errors: UploadError[];
  };
}
```

### 4.4 UM Consolidator API

#### GET /api/um-consolidator/reviews
```typescript
// Query Parameters
interface UMReviewsQuery {
  page?: number;
  limit?: number;
  status?: string;
  reviewerId?: string;
  participantId?: string;
  procedureCode?: string;
  requiresSecondaryReview?: boolean;
  startDate?: string;
  endDate?: string;
}

// Response (200 OK)
interface UMReviewsResponse {
  success: true;
  data: UMReview[];
  metadata: {
    total: number;
    page: number;
    limit: number;
  };
}
```

#### PUT /api/um-consolidator/reviews/:id
```typescript
// Request
interface UpdateUMReviewRequest {
  decision?: string;
  approvedHoursPerWeek?: number;
  reductionHours?: number;
  assessmentType?: string;
  notes?: string;
  status?: string;
  // Secondary review fields (role-restricted)
  secondaryReviewDecision?: string;
  updatedLtssHours?: number;
}

// Response (200 OK)
interface UpdateUMReviewResponse {
  success: true;
  data: UMReview;
}
```

#### POST /api/um-consolidator/reviews/:id/status
```typescript
// Request
interface UpdateStatusRequest {
  status: 'Assigned' | 'In Progress' | 'Secondary Review' | 'Review Completed';
  notes?: string;
}

// Response (200 OK)
interface UpdateStatusResponse {
  success: true;
  data: UMReview;
}
```

#### PUT /api/um-consolidator/reviews/:id/secondary-review
```typescript
// Request (Supervisor+ only)
interface SecondaryReviewRequest {
  secondaryReviewDecision: string;
  updatedLTSSHours?: number;
  secondaryReviewNotes?: string;
}

// Response (200 OK)
interface SecondaryReviewResponse {
  success: true;
  data: UMReview;
}

// Error Response (403 Forbidden)
interface SecondaryReviewForbiddenResponse {
  success: false;
  error: {
    code: 'FORBIDDEN';
    message: 'Secondary review access requires Supervisor role or higher';
  };
}
```

#### POST /api/um-consolidator/reviews/:id/calculate-hours
```typescript
// Request
interface CalculateHoursRequest {
  currentCarePlanHours: number;
  requestedHoursPerWeek: number;
  requestType: 'Increase' | 'Decrease' | 'Maintain';
}

// Response (200 OK)
interface CalculateHoursResponse {
  success: true;
  data: {
    hoursDifferenceCurrent: number;
    hoursDifferenceRequested: number;
    requiresSecondaryReview: boolean;
    secondaryReviewReasons: string[];
    calculationMetadata: {
      threshold: number;
      calculatedAt: string;
    };
  };
}
```

#### GET /api/um-consolidator/reviews/:id/audit-trail
```typescript
// Query Parameters
interface AuditTrailQuery {
  page?: number;
  limit?: number;  // Default: 50
}

// Response (200 OK)
interface AuditTrailResponse {
  success: true;
  data: {
    entries: UMAuditEntry[];
    total: number;
    page: number;
    limit: number;
  };
}

interface UMAuditEntry {
  id: string;
  umReviewId: string;
  userId: string;
  userName: string;
  action: 'CREATE' | 'UPDATE' | 'STATUS_CHANGE' | 'ASSIGN';
  fieldChanged?: string;
  oldValue?: string;
  newValue?: string;
  createdAt: Date;
}
```

#### POST /api/um-consolidator/assignments
```typescript
// Request (Supervisor+ only)
interface AssignCasesRequest {
  reviewIds: string[];
  reviewerId: string;
  procedureCode: string;
}

// Response (200 OK)
interface AssignCasesResponse {
  success: true;
  data: {
    assigned: number;
    failed: Array<{
      reviewId: string;
      reason: string;
    }>;
  };
}
```

#### POST /api/um-consolidator/assignments/bulk
```typescript
// Request (Supervisor+ only)
interface BulkAssignCasesRequest {
  reviewIds: string[];
  reviewerId: string;
  notes?: string;
}

// Response (200 OK)
interface BulkAssignCasesResponse {
  success: true;
  data: {
    assigned: number;
    failed: Array<{
      reviewId: string;
      reason: string;
    }>;
  };
}
```

#### GET /api/um-consolidator/assignments/stats
```typescript
// Response (200 OK) - Supervisor+ only
interface AssignmentStatsResponse {
  success: true;
  data: {
    totalPending: number;
    totalAssigned: number;
    totalCompleted: number;
    byReviewer: Array<{
      reviewerId: string;
      reviewerName: string;
      assignedCount: number;
      completedCount: number;
    }>;
    byProcedureCode: Record<string, number>;
  };
}
```

#### GET /api/um-consolidator/reviewers
```typescript
// Response (200 OK) - Supervisor+ only
interface ReviewersResponse {
  success: true;
  data: Array<{
    id: string;
    firstName: string;
    lastName: string;
    email: string;
    roles: string[];
    assignedCaseCount: number;
  }>;
}
```

#### PUT /api/um-consolidator/reviews/:id/assign
```typescript
// Request (Supervisor+ only)
interface QuickAssignRequest {
  reviewerId: string;
  notes?: string;
}

// Response (200 OK)
interface QuickAssignResponse {
  success: true;
  data: UMReview;
}
```

#### GET /api/um-consolidator/participants/search
```typescript
// Query Parameters
interface ParticipantSearchQuery {
  q: string;      // Minimum 2 characters
  limit?: number; // Default: 10
}

// Response (200 OK)
interface ParticipantSearchResponse {
  success: true;
  data: ParticipantLookupResult[];
}

interface ParticipantLookupResult {
  memberId: string;
  medicaidId: string;
  hcinId: string;
  firstName: string;
  lastName: string;
  dateOfBirth: Date;
  memberZone: string;
  planCategory: string;
}
```

### 4.5 Progress Notes API

#### GET /api/progress-notes/my-notes
```typescript
// Query Parameters
interface MyNotesQuery {
  page?: number;
  limit?: number;
  sortBy?: string;
  sortOrder?: 'asc' | 'desc';
  status?: 'draft' | 'submitted';
  query?: string;
}

// Response (200 OK)
interface MyNotesResponse {
  success: true;
  data: ProgressNotesInstance[];
  metadata: {
    total: number;
    draftCount: number;
    submittedCount: number;
  };
}
```

#### GET /api/progress-notes/team-notes
```typescript
// Query Parameters (Supervisor+ only)
interface TeamNotesQuery {
  page?: number;
  limit?: number;
  sortBy?: string;
  sortOrder?: 'asc' | 'desc';
  query?: string;
  reviewerId?: string;
}

// Response (200 OK)
interface TeamNotesResponse {
  success: true;
  data: ProgressNotesInstance[];
  metadata: {
    total: number;
    byReviewer: Record<string, number>;
  };
}
```

#### GET /api/progress-notes/supervisor-notes
```typescript
// Query Parameters (Supervisor+ only)
interface SupervisorNotesQuery {
  page?: number;
  limit?: number;
  sortBy?: string;
  sortOrder?: 'asc' | 'desc';
  status?: 'draft' | 'submitted';
  query?: string;
}

// Response (200 OK) - Combined view: own notes (all statuses) + team draft notes
interface SupervisorNotesResponse {
  success: true;
  data: ProgressNotesInstance[];
  metadata: {
    total: number;
    ownNotes: number;
    teamDraftNotes: number;
  };
}
```

#### GET /api/progress-notes/reviewers
```typescript
// Response (200 OK) - List of UM reviewers for reassignment
interface ReviewersResponse {
  success: true;
  data: {
    id: string;
    firstName: string;
    lastName: string;
    email: string;
    roles: string[];
  }[];
}
```

#### POST /api/progress-notes/instances
```typescript
// Request
interface CreateProgressNotesInstanceRequest {
  templateId?: string;           // Optional - auto-selected if not provided
  memberId?: string;
  providerId?: string;
  caseId?: string;
  questionResponses?: Record<string, unknown>;
  status?: 'draft' | 'submitted';
}

// Response (201 Created)
interface CreateProgressNotesInstanceResponse {
  success: true;
  data: ProgressNotesInstance;
}
```

#### PUT /api/progress-notes/instances/:id
```typescript
// Request
interface UpdateProgressNotesInstanceRequest {
  questionResponses?: Record<string, unknown>;
  generatedNotes?: Record<string, string>;
  status?: 'draft' | 'submitted';
}

// Response (200 OK)
interface UpdateProgressNotesInstanceResponse {
  success: true;
  data: ProgressNotesInstance;
}
```

#### POST /api/progress-notes/instances/:id/reassign
```typescript
// Request
interface ReassignNoteRequest {
  toReviewerId: string;
  reason?: string;
}

// Response (200 OK)
interface ReassignNoteResponse {
  success: true;
  data: {
    instance: ProgressNotesInstance;
    assignmentHistory: AssignmentHistoryEntry;
  };
}
```

#### GET /api/progress-notes/instances/:id/assignment-history
```typescript
// Response (200 OK)
interface AssignmentHistoryResponse {
  success: true;
  data: AssignmentHistoryEntry[];
}

interface AssignmentHistoryEntry {
  id: string;
  progressNoteId: string;
  fromReviewerId?: string;
  fromReviewerName?: string;
  toReviewerId: string;
  toReviewerName: string;
  assignedById: string;
  assignedByName: string;
  reason?: string;
  createdAt: Date;
}
```

#### POST /api/progress-notes/generate
```typescript
// Request
interface GenerateNotesRequest {
  responses: PredefinedQuestions;
}

interface PredefinedQuestions {
  // Member Information
  sentForReviewDate: string;
  age: number;
  sex: string;
  livingSituation: string;
  primaryDiagnoses: string;
  
  // Care Plan Information
  currentPlanOfCare: string;
  careplanBeingReviewed: string;
  careplanEffectiveDate: string;
  
  // Review Information
  additionalInfoRequested?: 'was' | 'was not';
  additionalInfoBeingRequested?: string;
  
  // Decision-specific fields
  approvedHours?: number;
  deniedHours?: number;
  reductionReason?: string;
  
  // Additional context
  [key: string]: unknown;
}

// Response (200 OK)
interface GenerateNotesResponse {
  success: true;
  data: {
    generatedNotes: Record<string, string>;  // Key: decision category, Value: generated note
    templateCount: number;
  };
}
```

#### GET /api/progress-notes/templates
```typescript
// Query Parameters
interface GetTemplatesQuery {
  category?: string;   // Filter by decision category
  isActive?: boolean;
}

// Response (200 OK)
interface GetTemplatesResponse {
  success: true;
  data: ProgressNotesTemplate[];
}

interface ProgressNotesTemplate {
  id: string;
  tenantId: string;
  name: string;
  description?: string;
  decisionCategory: string;
  noteTemplate: string;
  questions: Question[];
  predefinedTextConfig: Record<string, string[]>;
  version: number;
  isActive: boolean;
  createdAt: Date;
  updatedAt: Date;
}
```

#### POST /api/progress-notes/templates
```typescript
// Request (Manager+ only)
interface CreateTemplateRequest {
  name: string;
  description?: string;
  decisionCategory: string;
  noteTemplate: string;
  questions?: Question[];
  predefinedTextConfig?: Record<string, string[]>;
}

// Response (201 Created)
interface CreateTemplateResponse {
  success: true;
  data: ProgressNotesTemplate;
}
```

#### GET /api/progress-notes/templates/:id/history
```typescript
// Response (200 OK)
interface TemplateHistoryResponse {
  success: true;
  data: ProgressNotesTemplate[];  // All versions, ordered by version DESC
}
```

#### GET /api/progress-notes/analytics/dashboard
```typescript
// Response (200 OK) - Cached for 5 minutes
interface DashboardMetricsResponse {
  success: true;
  data: {
    totalNotes: number;
    draftCount: number;
    submittedCount: number;
    avgTimeSpentMinutes: number;
    notesThisWeek: number;
    notesThisMonth: number;
    templateUsage: Record<string, number>;
  };
}
```

#### GET /api/progress-notes/analytics/performance
```typescript
// Query Parameters
interface PerformanceQuery {
  reviewerId?: string;  // Optional - requires manager access for other reviewers
}

// Response (200 OK) - Cached for 10 minutes
interface PerformanceMetricsResponse {
  success: true;
  data: {
    reviewerId: string;
    reviewerName: string;
    totalNotes: number;
    avgTimeSpentMinutes: number;
    notesPerDay: number;
    templateBreakdown: Record<string, number>;
    trend: {
      period: string;
      count: number;
    }[];
  };
}
```

#### GET /api/progress-notes/analytics/trends
```typescript
// Query Parameters
interface TrendsQuery {
  days?: number;        // 1-365, default: 30
  reviewerId?: string;  // Optional filter
}

// Response (200 OK) - Cached for 30 minutes
interface TrendsResponse {
  success: true;
  data: {
    dailyTrend: {
      date: string;
      count: number;
      avgTimeMinutes: number;
    }[];
    weeklyTrend: {
      weekStart: string;
      count: number;
      avgTimeMinutes: number;
    }[];
    templateTrend: {
      category: string;
      counts: number[];
    }[];
  };
}

---

## 5. Security Design

### 5.1 JWT Token Structure

```typescript
// Access Token Payload
interface AccessTokenPayload {
  userId: string;
  tenantId: string;
  email: string;
  roles: string[];
  permissions: string[];
  iat: number;  // Issued at
  exp: number;  // Expiration (1 hour)
}

// Refresh Token Payload
interface RefreshTokenPayload {
  userId: string;
  tenantId: string;
  tokenId: string;  // Unique token identifier for revocation
  iat: number;
  exp: number;      // Expiration (7 days)
}
```

### 5.2 Role Permissions Matrix

| Permission | Admin | Manager | Supervisor | Coordinator | UM Nurse | QM Staff | UM Reviewer | UM Supervisor | UM Manager | UM Director |
|------------|-------|---------|------------|-------------|----------|----------|-------------|---------------|------------|-------------|
| users.read | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| users.write | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| members.read | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| members.write | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| forms.read | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| forms.write | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| forms.approve | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| um.read | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ (own) | ✅ | ✅ | ✅ |
| um.write | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ | ✅ (own) | ✅ | ✅ | ✅ |
| um.assign | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| um.secondary | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| um.audit | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ (own) | ✅ | ✅ | ✅ |
| progress.read | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| progress.write | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ |
| progress.templates | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| progress.analytics | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ | ✅ | ✅ | ✅ |
| progress.reassign | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| reports.read | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ | ✅ | ✅ | ✅ |
| admin.access | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |

**UM Role Hierarchy:**
- **UM Reviewer**: Can only view and update their own assigned cases
- **UM Supervisor**: Can view all cases, assign cases, access secondary review fields
- **UM Manager**: Same as Supervisor + template management
- **UM Director**: Full UM access including all analytics and reporting

### 5.3 Audit Trail Schema

```typescript
interface AuditLogEntry {
  id: string;
  tenantId: string;
  userId: string;
  action: 'CREATE' | 'READ' | 'UPDATE' | 'DELETE';
  resourceType: string;    // 'member', 'form', 'um_review', etc.
  resourceId: string;
  previousValue?: object;  // For updates
  newValue?: object;       // For creates/updates
  ipAddress: string;
  userAgent: string;
  timestamp: Date;
}
```

---

## 6. Integration Specifications

### 6.1 Frontend-Backend Integration

```typescript
// API Client Configuration
const API_BASE_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3001/api';

class ApiClient {
  private authToken: string | null = null;
  
  setAuthToken(token: string): void {
    this.authToken = token;
  }
  
  clearAuthToken(): void {
    this.authToken = null;
  }
  
  private async request<T>(
    endpoint: string, 
    options: RequestInit = {}
  ): Promise<ApiResponse<T>> {
    const headers: Record<string, string> = {
      'Content-Type': 'application/json',
      ...options.headers as Record<string, string>,
    };
    
    if (this.authToken) {
      headers['Authorization'] = `Bearer ${this.authToken}`;
    }
    
    const response = await fetch(`${API_BASE_URL}${endpoint}`, {
      ...options,
      headers,
    });
    
    if (!response.ok) {
      const errorData = await response.json().catch(() => ({}));
      throw new Error(errorData.message || `HTTP ${response.status}`);
    }
    
    return response.json();
  }
  
  async get<T>(endpoint: string, params?: Record<string, string>): Promise<ApiResponse<T>> {
    const queryString = params ? `?${new URLSearchParams(params)}` : '';
    return this.request<T>(`${endpoint}${queryString}`);
  }
  
  async post<T>(endpoint: string, data?: unknown): Promise<ApiResponse<T>> {
    return this.request<T>(endpoint, {
      method: 'POST',
      body: data ? JSON.stringify(data) : undefined,
    });
  }
  
  async put<T>(endpoint: string, data?: unknown): Promise<ApiResponse<T>> {
    return this.request<T>(endpoint, {
      method: 'PUT',
      body: data ? JSON.stringify(data) : undefined,
    });
  }
  
  async delete<T>(endpoint: string): Promise<ApiResponse<T>> {
    return this.request<T>(endpoint, { method: 'DELETE' });
  }
}

export const api = new ApiClient();
```

### 6.2 TanStack Query Integration

```typescript
// Query Client Configuration
export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,      // 5 minutes
      gcTime: 10 * 60 * 1000,        // 10 minutes (formerly cacheTime)
      retry: 3,
      retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
      refetchOnWindowFocus: false,
    },
    mutations: {
      retry: 1,
    },
  },
});

// Query Key Factory Pattern
export const queryKeys = {
  formCategories: {
    all: ['formCategories'] as const,
    lists: () => [...queryKeys.formCategories.all, 'list'] as const,
    list: (filters: string) => [...queryKeys.formCategories.lists(), { filters }] as const,
    details: () => [...queryKeys.formCategories.all, 'detail'] as const,
    detail: (id: string) => [...queryKeys.formCategories.details(), id] as const,
  },
  members: {
    all: ['members'] as const,
    search: (criteria: SearchCriteria) => [...queryKeys.members.all, 'search', criteria] as const,
    detail: (id: string) => [...queryKeys.members.all, 'detail', id] as const,
  },
  umReviews: {
    all: ['umReviews'] as const,
    list: (filters: UMReviewFilters) => [...queryKeys.umReviews.all, 'list', filters] as const,
    detail: (id: string) => [...queryKeys.umReviews.all, 'detail', id] as const,
  },
};
```

### 6.3 Error Handling Strategy

```typescript
// Frontend Error Boundary
class ErrorBoundary extends React.Component<Props, State> {
  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }
  
  componentDidCatch(error: Error, errorInfo: React.ErrorInfo): void {
    console.error('Error caught by boundary:', error, errorInfo);
    // Log to monitoring service
  }
  
  render(): React.ReactNode {
    if (this.state.hasError) {
      return <ErrorFallback error={this.state.error} />;
    }
    return this.props.children;
  }
}

// API Error Handler
function handleApiError(error: unknown): ApiResponse {
  if (error instanceof Error) {
    // Network errors
    if (error.message.includes('fetch')) {
      return {
        success: false,
        error: { code: 'NETWORK_ERROR', message: 'Unable to connect to server' }
      };
    }
    
    // Timeout errors
    if (error.message.includes('timeout')) {
      return {
        success: false,
        error: { code: 'TIMEOUT', message: 'Request timed out' }
      };
    }
  }
  
  return {
    success: false,
    error: { code: 'UNKNOWN_ERROR', message: 'An unexpected error occurred' }
  };
}
```

---

## 7. Performance Specifications

### 7.1 Frontend Bundle Targets

| Route Category | Target Size | Hard Limit | Dynamic Import Required |
|----------------|-------------|------------|------------------------|
| Core (dashboard, login) | <170kB | 200kB | No |
| Feature (forms, members) | <200kB | 300kB | Yes for heavy components |
| Admin | <200kB | 300kB | Yes |

### 7.2 Heavy Components Requiring Dynamic Import

```typescript
// Components >20kB must use dynamic imports
const FormBuilder = dynamic(() => import('@/components/features/form-builder'), {
  loading: () => <FormBuilderSkeleton />,
  ssr: false,
});

const AdvancedDataTable = dynamic(() => import('@/components/ui/data-table/advanced'), {
  loading: () => <TableSkeleton />,
  ssr: false,
});

const ChartDashboard = dynamic(() => import('@/components/charts/dashboard'), {
  loading: () => <ChartSkeleton />,
  ssr: false,
});

// Libraries requiring dynamic import:
// - @dnd-kit/* (~55kB)
// - @tanstack/react-table (~45kB)
// - recharts (~85kB)
```

### 7.3 Database Query Performance

```typescript
// Pagination Pattern (MANDATORY for lists)
async function getPaginatedResults<T>(
  table: string,
  tenantId: string,
  page: number,
  limit: number,
  orderBy: string = 'created_at DESC'
): Promise<PaginatedResult<T>> {
  const offset = (page - 1) * limit;
  
  // Count query (use indexed column)
  const countResult = await db.rawQuery(`
    SELECT COUNT(*) as total FROM ${table} WHERE tenant_id = @tenantId
  `, { tenantId });
  
  // Data query with OFFSET/FETCH
  const rows = await db.rawQuery(`
    SELECT * FROM ${table}
    WHERE tenant_id = @tenantId
    ORDER BY ${orderBy}
    OFFSET @offset ROWS FETCH NEXT @limit ROWS ONLY
  `, { tenantId, offset, limit });
  
  return {
    data: rows,
    total: countResult[0].total,
    page,
    limit,
    totalPages: Math.ceil(countResult[0].total / limit),
  };
}

// Existence Check Pattern (use EXISTS, not COUNT)
async function checkExists(table: string, conditions: object): Promise<boolean> {
  const whereClause = Object.keys(conditions)
    .map(key => `${key} = @${key}`)
    .join(' AND ');
  
  const result = await db.rawQuery(`
    SELECT CASE WHEN EXISTS (
      SELECT 1 FROM ${table} WHERE ${whereClause}
    ) THEN 1 ELSE 0 END as exists
  `, conditions);
  
  return result[0].exists === 1;
}
```

### 7.4 Caching Strategy

```typescript
// Redis Cache Service
class CacheService {
  private redis: Redis;
  private defaultTTL = 300; // 5 minutes
  
  async get<T>(key: string): Promise<T | null> {
    const cached = await this.redis.get(key);
    return cached ? JSON.parse(cached) : null;
  }
  
  async set<T>(key: string, value: T, ttl?: number): Promise<void> {
    await this.redis.setex(key, ttl || this.defaultTTL, JSON.stringify(value));
  }
  
  async invalidate(pattern: string): Promise<void> {
    const keys = await this.redis.keys(pattern);
    if (keys.length > 0) {
      await this.redis.del(...keys);
    }
  }
  
  // Cache key patterns
  static keys = {
    formCategories: (tenantId: string) => `form:categories:${tenantId}`,
    formTemplate: (id: string) => `form:template:${id}`,
    memberSearch: (tenantId: string, query: string) => `member:search:${tenantId}:${query}`,
    userSession: (userId: string) => `session:${userId}`,
  };
}
```

---

## 8. Deployment Specifications

### 8.1 Docker Configuration

#### Frontend Dockerfile (Optimized - 203MB)
```dockerfile
# Build stage
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production=false
COPY . .
RUN npm run build

# Runtime stage
FROM node:18-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production

# Create non-root user
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

# Copy standalone build
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
COPY --from=builder --chown=nextjs:nodejs /app/public ./public

USER nextjs
EXPOSE 3000
ENV PORT=3000
CMD ["node", "server.js"]
```

#### Backend Dockerfile (Optimized - 335MB)
```dockerfile
# Build stage
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Runtime stage
FROM node:18-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production

# Create non-root user
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 expressjs

# Copy built application
COPY --from=builder --chown=expressjs:nodejs /app/dist ./dist
COPY --from=builder --chown=expressjs:nodejs /app/package*.json ./
RUN npm ci --only=production

USER expressjs
EXPOSE 3001
CMD ["node", "dist/index.js"]
```

### 8.2 Docker Compose Configuration

#### Infrastructure (docker-compose.yml)
```yaml
version: '3.8'

services:
  mssql:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: chc-crm-infrastructure-mssql
    environment:
      - ACCEPT_EULA=Y
      - MSSQL_SA_PASSWORD=${DATABASE_PASSWORD}
      - MSSQL_PID=Developer
    ports:
      - "1433:1433"
    volumes:
      - mssql-data:/var/opt/mssql
    healthcheck:
      test: /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P "${DATABASE_PASSWORD}" -Q "SELECT 1"
      interval: 10s
      timeout: 5s
      retries: 10
    networks:
      - chc-crm-infrastructure-network

  redis:
    image: redis:7-alpine
    container_name: chc-crm-infrastructure-redis
    command: redis-server --requirepass ${REDIS_PASSWORD}
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    healthcheck:
      test: redis-cli -a ${REDIS_PASSWORD} ping
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - chc-crm-infrastructure-network

volumes:
  mssql-data:
  redis-data:

networks:
  chc-crm-infrastructure-network:
    driver: bridge
```

#### Application (per environment)
```yaml
version: '3.8'

services:
  frontend:
    build:
      context: ../../frontend
      dockerfile: Dockerfile
    container_name: chc-crm-${ENVIRONMENT}-frontend
    environment:
      - NODE_ENV=production
      - NEXT_PUBLIC_API_URL=http://backend:3001/api
    ports:
      - "${FRONTEND_PORT}:3000"
    depends_on:
      - backend
    networks:
      - chc-crm-${ENVIRONMENT}-network
      - chc-crm-infrastructure-network

  backend:
    build:
      context: ../../backend
      dockerfile: Dockerfile
    container_name: chc-crm-${ENVIRONMENT}-backend
    environment:
      - NODE_ENV=production
      - APP_ENVIRONMENT=${ENVIRONMENT}
      - DATABASE_HOST=chc-crm-infrastructure-mssql
      - DATABASE_NAME=${DATABASE_NAME}
      - DATABASE_PASSWORD=${DATABASE_PASSWORD}
      - REDIS_HOST=chc-crm-infrastructure-redis
      - REDIS_PASSWORD=${REDIS_PASSWORD}
      - REDIS_DB=${REDIS_DB}
      - JWT_SECRET=${JWT_SECRET}
    ports:
      - "${BACKEND_PORT}:3001"
    networks:
      - chc-crm-${ENVIRONMENT}-network
      - chc-crm-infrastructure-network

networks:
  chc-crm-${ENVIRONMENT}-network:
    driver: bridge
  chc-crm-infrastructure-network:
    external: true
```

### 8.3 Environment Variables

```bash
# Infrastructure (.env.infrastructure)
DATABASE_PASSWORD=YourStrongPassword123!
REDIS_PASSWORD=YourRedisPassword123!

# Development (.env.development)
ENVIRONMENT=development
DATABASE_NAME=chc_insight_development
REDIS_DB=0
FRONTEND_PORT=3000
BACKEND_PORT=3001
JWT_SECRET=dev-jwt-secret-change-in-production

# Staging (.env.staging)
ENVIRONMENT=staging
DATABASE_NAME=chc_insight_staging
REDIS_DB=1
FRONTEND_PORT=3010
BACKEND_PORT=3011
JWT_SECRET=staging-jwt-secret

# Production (.env.production)
ENVIRONMENT=production
DATABASE_NAME=chc_insight_production
REDIS_DB=2
FRONTEND_PORT=3020
BACKEND_PORT=3021
JWT_SECRET=${PRODUCTION_JWT_SECRET}  # From secrets manager
```

---

## 9. Testing Specifications

### 9.1 Unit Testing

```typescript
// Service Unit Test Example
describe('FormHierarchyService', () => {
  let service: FormHierarchyService;
  let mockDb: jest.Mocked<UniversalDatabaseService>;
  
  beforeEach(() => {
    mockDb = {
      rawQuery: jest.fn(),
      insert: jest.fn(),
      update: jest.fn(),
      delete: jest.fn(),
    } as any;
    
    service = new FormHierarchyService();
    (service as any).db = mockDb;
  });
  
  describe('getCategoryById', () => {
    it('should return category when found', async () => {
      mockDb.rawQuery.mockResolvedValue([{
        id: 'cat-1',
        name: 'Cases',
        tenant_id: 'tenant-1',
      }]);
      
      const result = await service.getCategoryById('cat-1', 'tenant-1');
      
      expect(result.success).toBe(true);
      expect(result.data?.name).toBe('Cases');
    });
    
    it('should return NOT_FOUND when category does not exist', async () => {
      mockDb.rawQuery.mockResolvedValue([]);
      
      const result = await service.getCategoryById('invalid', 'tenant-1');
      
      expect(result.success).toBe(false);
      expect(result.error?.code).toBe('NOT_FOUND');
    });
  });
});
```

### 9.2 Integration Testing

```typescript
// API Integration Test Example
describe('POST /api/form-categories', () => {
  let authToken: string;
  
  beforeAll(async () => {
    // Login to get auth token
    const loginResponse = await request(app)
      .post('/api/auth/login')
      .send({ email: 'admin@test.com', password: 'password' });
    authToken = loginResponse.body.data.accessToken;
  });
  
  it('should create a new category', async () => {
    const response = await request(app)
      .post('/api/form-categories')
      .set('Authorization', `Bearer ${authToken}`)
      .send({ name: 'Test Category', description: 'Test description' });
    
    expect(response.status).toBe(201);
    expect(response.body.success).toBe(true);
    expect(response.body.data.name).toBe('Test Category');
  });
  
  it('should return 401 without auth token', async () => {
    const response = await request(app)
      .post('/api/form-categories')
      .send({ name: 'Test Category' });
    
    expect(response.status).toBe(401);
  });
  
  it('should return 400 for invalid data', async () => {
    const response = await request(app)
      .post('/api/form-categories')
      .set('Authorization', `Bearer ${authToken}`)
      .send({ name: '' });  // Empty name
    
    expect(response.status).toBe(400);
    expect(response.body.error.code).toBe('VALIDATION_ERROR');
  });
});
```

---

## 10. Appendices

### Appendix A: Type Definitions

```typescript
// Core Types
interface User {
  id: string;
  email: string;
  firstName: string;
  lastName: string;
  roles: Role[];
  tenantId: string;
  isActive: boolean;
}

interface Role {
  id: string;
  name: string;
  permissions: Permission[];
}

interface Permission {
  resource: string;
  actions: ('create' | 'read' | 'update' | 'delete')[];
}

// Form Hierarchy Types
interface FormCategory {
  id: string;
  name: string;
  description?: string;
  tenantId: string;
  isActive: boolean;
  createdAt: Date;
  updatedAt: Date;
}

interface FormType {
  id: string;
  categoryId: string;
  name: string;
  description?: string;
  businessRules: BusinessRule[];
  tenantId: string;
  isActive: boolean;
}

interface FormTemplate {
  id: string;
  typeId: string;
  name: string;
  version: number;
  templateData: TemplateData;
  workflowConfig: WorkflowConfig;
  effectiveDate: Date;
  expirationDate?: Date;
}

interface FormInstance {
  id: string;
  templateId: string;
  memberId?: string;
  providerId?: string;
  assignedTo?: string;
  status: FormStatus;
  responseData: ResponseData[];
  dueDate?: Date;
  submittedAt?: Date;
  approvedAt?: Date;
}

type FormStatus = 'draft' | 'in_progress' | 'submitted' | 'approved' | 'rejected' | 'cancelled';

// UM Consolidator Types
interface UMReview {
  id: string;
  tenantId: string;
  participantId: string;
  participantFirstName?: string;
  participantLastName?: string;
  participantZone?: string;
  reviewerId?: string;
  reviewerName?: string;
  procedureCode: string;
  currentCarePlanHours: number;
  requestType: 'Increase' | 'Decrease' | 'Maintain';
  requestedHoursPerWeek: number;
  approvedHoursPerWeek?: number;
  decision?: UMDecision;
  reductionHours?: number;
  assessmentType?: 'F2F Assessment' | 'Telephonic Assessment';
  psstHours?: number;
  requiresSecondaryReview: boolean;
  hoursDifferenceCurrent?: number;
  hoursDifferenceRequested?: number;
  secondaryReviewDecision?: UMDecision;
  updatedLtssHours?: number;
  updatedDifference?: number;
  status: UMReviewStatus;
  notes?: string;
  secondaryReviewNotes?: string;  // Notes from secondary review
  reviewDate: Date;
  weekOfYear?: number;
  createdBy: string;
  updatedBy?: string;
  createdAt: Date;
  updatedAt: Date;
}

type UMReviewStatus = 'Assigned' | 'In Progress' | 'Secondary Review' | 'Review Completed';

type UMDecision = 
  | 'Approval' 
  | 'Denial' 
  | 'Denial - With Reduction' 
  | 'Denial - Would have Reduced' 
  | 'Flagged reduction' 
  | 'More information' 
  | 'Reduction';

interface Appeal {
  id: string;
  tenantId: string;
  umReviewId: string;
  participantId: string;
  appealReviewerId?: string;
  appealHearing: Date;
  deniedHours: number;
  appealType: AppealType;
  appealResults?: AppealResult;
  hoursAfterGrievance?: number;
  reasonForPasHours?: string;
  employeeVoterId?: string;
  medicalDirectorId?: string;
  umReviewerId?: string;
  remediationEffectiveDate?: Date;
  status: AppealStatus;
  yearOfReview?: number;
  createdBy: string;
  updatedBy?: string;
  createdAt: Date;
  updatedAt: Date;
}

type AppealType = 
  | 'Deny Increase/Reduction' 
  | 'DME' 
  | 'Home Adaptation' 
  | 'Increase Request' 
  | 'Initial Request' 
  | 'Reduction';

type AppealResult = 
  | 'Overturned' 
  | 'Partially Overturned' 
  | 'Removed From Schedule' 
  | 'Upheld' 
  | 'Withdrawn';

type AppealStatus = 'Submitted' | 'Hearing' | 'Decision';

interface UMAuditEntry {
  id: string;
  umReviewId: string;
  userId: string;
  userName?: string;
  action: 'CREATE' | 'UPDATE' | 'STATUS_CHANGE' | 'ASSIGN';
  fieldChanged?: string;
  oldValue?: string;
  newValue?: string;
  createdAt: Date;
}

// Progress Notes Types
interface ProgressNotesTemplate {
  id: string;
  tenantId: string;
  name: string;
  description?: string;
  decisionCategory: ProgressNotesDecisionCategory;
  noteTemplate: string;
  questions: Question[];
  predefinedTextConfig: Record<string, string[]>;
  version: number;
  isActive: boolean;
  createdBy: string;
  updatedBy?: string;
  createdAt: Date;
  updatedAt: Date;
}

type ProgressNotesDecisionCategory = 
  | 'Request for Additional Information'
  | 'Approval to continue current plan of care'
  | 'Approval of increase'
  | 'Complete Denial'
  | 'Partially Deny an Increase'
  | 'Reduction of Current plan of care'
  | 'Denial of Increase & Reduction of current plan of care';

interface ProgressNotesInstance {
  id: string;
  tenantId: string;
  templateId: string;
  template?: ProgressNotesTemplate;
  reviewerId: string;
  reviewerName?: string;
  memberId?: string;
  memberName?: string;
  providerId?: string;
  providerName?: string;
  caseId?: string;
  carePlanReviewType?: 'increase' | 'decrease' | 'continue';  // Care plan review type
  questionResponses: Record<string, unknown>;
  generatedNotes: Record<string, string>;
  status: ProgressNotesStatus;
  submittedAt?: Date;
  startedAt?: Date;
  completedAt?: Date;
  timeSpentMinutes?: number;
  createdBy: string;
  updatedBy?: string;
  createdAt: Date;
  updatedAt: Date;
}

type ProgressNotesStatus = 'draft' | 'submitted';

interface ProgressNotesQuestionTemplate {
  id: string;
  tenantId: string;
  version: number;
  questionsConfig: PredefinedQuestionsConfig;
  isActive: boolean;
  createdBy: string;
  createdAt: Date;
  updatedAt: Date;
}

interface PredefinedQuestionsConfig {
  sections: QuestionSection[];
  questions: PredefinedQuestion[];
}

interface QuestionSection {
  id: string;
  title: string;
  description?: string;
  order: number;
}

interface PredefinedQuestion {
  id: string;
  sectionId: string;
  label: string;
  type: 'text' | 'number' | 'date' | 'select' | 'multiselect' | 'textarea';
  required: boolean;
  options?: string[];
  placeholder?: string;
  helpText?: string;
  order: number;
}

interface ProgressNotesAssignmentHistory {
  id: string;
  progressNoteId: string;
  fromReviewerId?: string;
  fromReviewerName?: string;
  toReviewerId: string;
  toReviewerName?: string;
  assignedBy: string;
  assignedByName?: string;
  reason?: string;
  createdAt: Date;
}

interface ProgressNotesAnalytics {
  id: string;
  tenantId: string;
  reviewerId: string;
  instanceId: string;                         // Reference to progress_notes_instances
  templateTypesGenerated?: string;            // JSON: template types used
  completionTimeMinutes?: number;             // Time to complete the note
  questionsAnswered?: number;                 // Number of questions answered
  notesCopiedCount: number;                   // Number of times notes were copied
  carePlanReviewType?: 'increase' | 'decrease' | 'continue';
  createdAt: Date;
}

interface ProgressNotesMonthlySum {
  id: string;
  tenantId: string;
  reviewerId: string;
  year: number;
  month: number;
  totalNotes: number;
  completedNotes: number;
  avgCompletionTimeMinutes?: number;
  totalTemplateTypesUsed: number;
  mostUsedTemplateType?: string;
  createdAt: Date;
}

interface DashboardMetrics {
  totalNotes: number;
  draftNotes: number;
  completedNotes: number;
  thisMonthNotes: number;
  thisWeekNotes: number;
  recentNotes: ProgressNotesInstance[];
  performanceMetrics: {
    averageCompletionTime: number;
    notesPerDay: number;
    carePlanReviewTypeStats: Record<string, number>;
    completionTrends?: TrendDataPoint[];
  };
}

interface PerformanceMetrics {
  averageCompletionTime: number;
  notesPerDay: number;
  carePlanReviewTypeStats: Record<string, number>;
  completionTrends?: TrendDataPoint[];
}
}

interface TrendDataPoint {
  period: string;
  count: number;
  avgTimeMinutes?: number;
}

// API Types
interface ApiResponse<T = unknown> {
  success: boolean;
  data?: T;
  error?: ApiError;
  metadata?: ResponseMetadata;
}

interface ApiError {
  code: string;
  message: string;
  details?: unknown;
}

interface ResponseMetadata {
  total?: number;
  page?: number;
  limit?: number;
  timestamp: string;
}

interface PaginatedResult<T> {
  data: T[];
  total: number;
  page: number;
  limit: number;
  totalPages: number;
}
```

### Appendix B: Error Codes Reference

| Code | HTTP Status | Description |
|------|-------------|-------------|
| UNAUTHORIZED | 401 | Authentication required or token invalid |
| FORBIDDEN | 403 | Insufficient permissions |
| NOT_FOUND | 404 | Resource not found |
| VALIDATION_ERROR | 400 | Request validation failed |
| ALREADY_EXISTS | 409 | Duplicate resource |
| TEMPLATE_NOT_FOUND | 422 | Referenced template missing |
| DATABASE_ERROR | 500 | Database operation failed |
| NETWORK_ERROR | 503 | Unable to connect to service |
| TIMEOUT | 504 | Request timed out |

#### UM Consolidator Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| INVALID_TRANSITION | 400 | Invalid status transition (e.g., Assigned → Review Completed) |
| ASSIGNMENT_ERROR | 400 | Case assignment failed |
| PARTICIPANT_NOT_FOUND | 404 | Participant not found in member staging |
| INVALID_REVIEWER | 400 | Reviewer not found or invalid role |
| INVALID_PROCEDURE_CODE | 400 | Invalid procedure code |
| INVALID_OPERATION | 400 | Invalid operation for current review state |
| CALCULATION_ERROR | 500 | Hour calculation operation failed |
| AUDIT_ERROR | 500 | Audit logging failed |
| AUDIT_RETRIEVAL_ERROR | 500 | Failed to retrieve audit trail |
| SECONDARY_REVIEW_FORBIDDEN | 403 | Secondary review access requires Supervisor+ role |

#### Progress Notes Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| NOTE_GENERATION_ERROR | 500 | Failed to generate progress notes |
| TEMPLATE_VALIDATION_ERROR | 400 | Template validation failed |
| INSUFFICIENT_PERMISSIONS | 403 | Role-based access denied |
| REASSIGNMENT_ERROR | 400 | Note reassignment failed |
| ANALYTICS_ERROR | 500 | Analytics calculation failed |
| INVALID_DECISION_CATEGORY | 400 | Invalid decision category for template |
| QUESTION_VALIDATION_ERROR | 400 | Invalid question responses |

---

**Document Version:** 1.1.0  
**Last Updated:** February 2026  
**Author:** CHC Insight Development Team
