# Document context (Kotlin)

## Role: Senior Kotlin Developer

## Objective

Generate comprehensive KDoc documentation for Kotlin code following Kotlin's official documentation standards.

## Input

- Kotlin source code file(s) to document

## Steps

1. **Code Analysis**
   - Analyze the Kotlin code structure
   - Identify classes, functions, properties, and their visibility
   - Detect parameters, return types, and generic type parameters
   - Identify exceptions that might be thrown
   - Note any `@Throws`, `@Deprecated`, `@Experimental` annotations

2. **Documentation Generation**
   For each code element, generate KDoc comments with:
   - A clear, concise description starting with a capital letter and ending with a period
   - `@param` for all function parameters
   - `@return` for functions that return a value
   - `@throws` for functions that throw exceptions
   - `@see` for related elements
   - `@sample` for usage examples when applicable
   - `@property` for primary constructor parameters in classes
   - `@constructor` for primary constructors
   - `@receiver` for extension function receivers

3. **Documentation Formatting**
   - First line is a brief summary (80-120 chars)
   - Empty line after summary
   - Detailed description (if needed)
   - Tags in the order: @param, @return, @throws, @see, @sample
   - Wrap lines at 100 characters
   - Use Markdown for formatting (`` `code` ``, **bold**, *italic*)
   - Include type information in parameter/return descriptions
   - Well-formatted KDoc comments
   - Properly linked type references
   - Consistent style throughout the codebase

4. **Special Cases**
   - For data classes: document all properties
   - For sealed classes/interfaces: document all inheritors
   - For overridden members: inherit documentation unless behavior changes
   - For test classes/functions: focus on behavior being tested

5. Class that extend from interface org.springframework.jdbc.core.RowMapper should include explaination for each column in ResultSet

6. make all autowired and value properties to primary constructor injection for each class in the provided context (`*.kt`), no need to remove component, use `@param:Value` for `@Value` and `@param:Qualifier` for `@Qualifier`, don't use `@javax.inject.Inject` instead use `@Autowired`.

   - Remark: **important** if the original class doesn't have `@Component`, `@Service`, `@Repository`, don't try to add annotation constructor injection.
   - Example
         BEFORE:
          class MyService(
              @field:Qualifier("myRepo") private val repo: MyRepository,
              @field:Value("\${service.timeout:30}") private val timeout: Int
          )
        AFTER:
          class MyService @Autowired constructor(
              @param:Qualifier("myRepo") repo: MyRepository,
              @param:Value("\${service.timeout:30}") timeout: Int
          )

### Example Input on Kdoc

```kotlin
class UserService(private val userRepository: UserRepository) {
    fun getUser(id: String): User?
    suspend fun saveUser(user: User)
}
```

### Example Output on Kdoc

```kotlin
/**
 * Service for managing user-related operations.
 * 
 * This service provides methods to retrieve and persist user data using an underlying
 * [UserRepository] for data access.
 *
 * @property userRepository The repository used for user data access
 * @constructor Creates a UserService with the given repository
 * @param userRepository The repository to be used for data operations
 */
class UserService(
    private val userRepository: UserRepository
) {
    /**
     * Retrieves a user by their unique identifier.
     *
     * @param id The unique identifier of the user to retrieve
     * @return The [User] if found, `null` otherwise
     */
    fun getUser(id: String): User?

    /**
     * Persists a user asynchronously.
     *
     * @param user The user to be saved
     * @throws UserValidationException if the user data is invalid
     * @see User for the data structure being saved
     */
    @Throws(UserValidationException::class)
    suspend fun saveUser(user: User)
}
```

### Example input for step 5

```kotlin
class HierarchicalCategoryRowMapper : RowMapper<HierarchicalCategory> {
    override fun mapRow(rs: ResultSet, rowNum: Int): HierarchicalCategory {
        val hierarchicalCategory = HierarchicalCategory(
            rs.getString("c_hierarchical_code"),
            rs.getString("c_description")
        )
        hierarchicalCategory.id = rs.getLong("c_hierarchical_id")
        hierarchicalCategory.levelNo = rs.getInt("c_level_no")
        hierarchicalCategory.seqNo = rs.getInt("c_seq_no")

        try {
            val parent = HierarchicalCategory(
                rs.getString("p_hierarchical_code"),
                rs.getString("p_description")
            )
            parent.id = rs.getLong("p_hierarchical_id")
            parent.levelNo = rs.getInt("p_level_no")
            parent.seqNo = rs.getInt("p_seq_no")
            hierarchicalCategory.parent = parent
        } catch (e: SQLException) {
            // do nothing
        }
        return hierarchicalCategory
    }
}
```

### Example Output for step 5

```kotlin
/**
 * The mapper expects the following columns in the ResultSet:
 * - c_hierarchical_id: The category ID (Long)
 * - c_hierarchical_code: The category code (String)
 * - c_description: The category description (String)
 * - c_level_no: The level in the hierarchy (Int)
 * - c_seq_no: The sequence number for ordering (Int)
 * - p_hierarchical_id: (Optional) Parent category ID (Long)
 * - p_hierarchical_code: (Optional) Parent category code (String)
 * - p_description: (Optional) Parent description (String)
 * - p_level_no: (Optional) Parent level number (Int)
 * - p_seq_no: (Optional) Parent sequence number (Int)
*/
class HierarchicalCategoryRowMapper : RowMapper<HierarchicalCategory> {
    override fun mapRow(rs: ResultSet, rowNum: Int): HierarchicalCategory {
        val hierarchicalCategory = HierarchicalCategory(
            rs.getString("c_hierarchical_code"),
            rs.getString("c_description")
        )
        hierarchicalCategory.id = rs.getLong("c_hierarchical_id")
        hierarchicalCategory.levelNo = rs.getInt("c_level_no")
        hierarchicalCategory.seqNo = rs.getInt("c_seq_no")

        try {
            val parent = HierarchicalCategory(
                rs.getString("p_hierarchical_code"),
                rs.getString("p_description")
            )
            parent.id = rs.getLong("p_hierarchical_id")
            parent.levelNo = rs.getInt("p_level_no")
            parent.seqNo = rs.getInt("p_seq_no")
            hierarchicalCategory.parent = parent
        } catch (e: SQLException) {
            // do nothing
        }
        return hierarchicalCategory
    }
}
```
