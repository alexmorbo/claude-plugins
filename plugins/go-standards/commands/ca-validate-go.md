Validate Go service for Clean Architecture compliance

Perform comprehensive Clean Architecture validation for a Go service:

## 🎯 Usage
**Command**: `/ca-validate-go [service-path]`
**Example**: `/ca-validate-go auth-service`

## 🔍 What it validates

### 1. Directory Structure
- ✅ Verify required CA directories exist (domain/, application/, infrastructure/, interface/, cmd/)
- ✅ Check proper subdirectory organization (entity/, usecase/, etc.)
- ✅ Validate Go module structure (go.mod, go.sum)

### 2. Dependency Direction
- ✅ Domain layer has no external dependencies
- ✅ Application layer only imports domain
- ✅ Infrastructure implements domain interfaces
- ✅ Interface layer is thin and uses application layer

### 3. Domain Layer Purity
- ✅ No database imports (database/sql, gorm.io)
- ✅ No HTTP framework imports (gin-gonic, gorilla/mux)
- ✅ No infrastructure dependencies
- ✅ Pure business logic only

### 4. Code Quality
- ✅ Use cases follow single responsibility principle
- ✅ Repository interfaces properly defined
- ✅ No business logic in HTTP handlers
- ✅ Proper error handling patterns

## 📊 Validation Process

1. **Analyze Project Structure**: Check directory layout and Go modules
2. **Scan Dependencies**: Parse import statements and go.mod
3. **Validate Layer Separation**: Ensure proper CA boundaries
4. **Check Code Patterns**: Verify use cases, repositories, handlers
5. **Generate Report**: Provide detailed findings and suggestions

## 📋 Example Output

```
🎯 Clean Architecture Validation Report for auth-service

✅ PASSED: Directory structure is CA-compliant
✅ PASSED: Dependencies follow inward direction
✅ PASSED: Domain layer is pure (no external deps)
❌ FAILED: Found business logic in HTTP handler
❌ FAILED: Domain service has direct database import

📊 Overall Score: 67/100 (Needs improvement)

🔧 Suggested Fixes:
1. Move business logic from interface/http/auth_handler.go to application/usecase/
2. Remove "database/sql" import from domain/service/user_service.go
3. Create repository interface in domain/repository/user_repository.go

🚀 Next Steps:
- Apply suggested refactoring
- Review domain layer design
- Ensure all use cases are properly tested
```

## 🛠️ Technical Implementation

This command analyzes:
- **File Structure**: Checks for CA directory organization
- **Import Analysis**: Scans Go import statements for violations
- **Code Patterns**: Validates architectural patterns and conventions
- **Dependency Graph**: Maps module dependencies using go mod graph

## 🔗 Related Commands

- `/ca-validate-frontend` - Validate Next.js applications
- `/ca-init-go` - Create new Go service with CA structure
- Use after making changes to verify compliance

## 📚 More Information

See: `homelab-ci/documentation/ca/slash-commands-ca.md` for detailed documentation.