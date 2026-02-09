# JHipster Backend Development Skill

## Description
Use this skill when working with JHipster backend development: creating entities, services, repositories, REST controllers, database migrations (Liquibase), security configurations, DTOs/mappers, custom queries, and Spring Boot configurations. Triggers include: mentions of JHipster, Spring Boot microservices, entity generation, service layer implementation, JWT authentication, or database schema changes.

## Core Patterns

### 1. Entity Creation
- Always follow JHipster conventions: `@Entity`, `@Table`, `AbstractAuditingEntity` for audit fields
- Use proper relationships: `@ManyToOne`, `@OneToMany` with `@JoinColumn`
- Include validation annotations: `@NotNull`, `@Size`, `@Pattern`
- Implement `Serializable` and generate proper `equals()` and `hashCode()`

### 2. Repository Layer
- Extend `JpaRepository<Entity, Long>`
- Use Spring Data JPA query methods
- For complex queries, use `@Query` with JPQL
- Include custom repository methods when needed
- Use `Specification` pattern for dynamic filters

### 3. Service Layer Architecture
```java
// Service Interface
public interface EntityService {
    EntityDTO save(EntityDTO dto);
    Optional<EntityDTO> findOne(Long id);
    Page<EntityDTO> findAll(Pageable pageable);
    void delete(Long id);
}

// Service Implementation
@Service
@Transactional
public class EntityServiceImpl implements EntityService {
    private final EntityRepository repository;
    private final EntityMapper mapper;
    
    // Constructor injection (JHipster standard)
    public EntityServiceImpl(EntityRepository repository, EntityMapper mapper) {
        this.repository = repository;
        this.mapper = mapper;
    }
}
```

### 4. REST Controller Pattern
```java
@RestController
@RequestMapping("/api")
public class EntityResource {
    
    private final Logger log = LoggerFactory.getLogger(EntityResource.class);
    private final EntityService entityService;
    
    // Standard CRUD endpoints
    @PostMapping("/entities")
    public ResponseEntity<EntityDTO> createEntity(@Valid @RequestBody EntityDTO dto) 
        throws URISyntaxException {
        
        log.debug("REST request to save Entity : {}", dto);
        if (dto.getId() != null) {
            throw new BadRequestAlertException("A new entity cannot already have an ID", 
                ENTITY_NAME, "idexists");
        }
        EntityDTO result = entityService.save(dto);
        return ResponseEntity.created(new URI("/api/entities/" + result.getId()))
            .headers(HeaderUtil.createEntityCreationAlert(applicationName, true, 
                ENTITY_NAME, result.getId().toString()))
            .body(result);
    }
    
    @GetMapping("/entities")
    public ResponseEntity<List<EntityDTO>> getAllEntities(Pageable pageable) {
        // Implementation
    }
}
```

### 5. DTO & Mapper Pattern
```java
// DTO
public class EntityDTO implements Serializable {
    private Long id;
    @NotNull
    private String name;
    // Getters/Setters, equals, hashCode
}

// Mapper using MapStruct
@Mapper(componentModel = "spring", uses = {RelatedEntityMapper.class})
public interface EntityMapper extends BaseEntityMapper<EntityDTO, Entity> {
    
    @Mapping(source = "relatedEntity.id", target = "relatedEntityId")
    @Mapping(source = "relatedEntity.name", target = "relatedEntityName")
    EntityDTO toDto(Entity entity);
    
    @Mapping(source = "relatedEntityId", target = "relatedEntity")
    Entity toEntity(EntityDTO dto);
}
```

### 6. Liquibase Migrations
```xml
<!-- changelog/[timestamp]_added_entity_Entity.xml -->
<changeSet id="[timestamp]-1" author="jhipster">
    <createTable tableName="entity">
        <column name="id" type="bigint" autoIncrement="true">
            <constraints primaryKey="true" nullable="false"/>
        </column>
        <column name="name" type="varchar(255)">
            <constraints nullable="false"/>
        </column>
        <column name="created_by" type="varchar(50)">
            <constraints nullable="false"/>
        </column>
        <column name="created_date" type="timestamp"/>
        <column name="last_modified_by" type="varchar(50)"/>
        <column name="last_modified_date" type="timestamp"/>
    </createTable>
</changeSet>
```

### 7. Security Configuration
- Use `@PreAuthorize("hasAuthority('ROLE_USER')")` for method security
- Configure `SecurityConfiguration.java` for endpoint protection
- Implement custom `UserDetailsService` when needed
- Use JWT tokens for stateless authentication

### 8. Exception Handling
```java
// Use JHipster's exception patterns
throw new BadRequestAlertException("Error message", "entityName", "errorKey");
throw new EntityNotFoundException("Entity not found");

// Global exception handler already provided by JHipster
```

## Best Practices

### Database
- Always use Liquibase for schema changes (never manual SQL)
- Include rollback changesets
- Use meaningful changelog IDs: `YYYYMMDDHHMMSS-sequence`
- Test migrations on clean database

### Performance
- Use `@Transactional(readOnly = true)` for read operations
- Implement pagination for list endpoints
- Use DTOs to avoid lazy loading issues
- Consider caching with `@Cacheable` for frequent queries

### Testing
```java
@SpringBootTest(classes = YourApp.class)
@AutoConfigureMockMvc
@WithMockUser
class EntityResourceIT {
    
    @Autowired
    private EntityRepository repository;
    
    @Autowired
    private EntityMapper mapper;
    
    @Autowired
    private MockMvc restEntityMockMvc;
    
    @Test
    @Transactional
    void createEntity() throws Exception {
        // Integration test implementation
    }
}
```

### Code Organization
- Entities in `domain/` package
- DTOs in `service/dto/` package
- Mappers in `service/mapper/` package
- Services in `service/` package
- Repositories in `repository/` package
- Controllers in `web/rest/` package

## Common Issues & Solutions

### Issue: LazyInitializationException
**Solution**: Use DTOs with proper mapping, or use `@EntityGraph` for fetch joins

### Issue: Circular dependencies in mappers
**Solution**: Use `@Mapper(uses = {})` carefully and break cycles with custom mapping methods

### Issue: Database collation problems (MySQL to SQL Server)
**Solution**: Specify collation in Liquibase: `<column collation="SQL_Latin1_General_CP1_CI_AS">`

### Issue: Audit fields not populating
**Solution**: Ensure entity extends `AbstractAuditingEntity` and `@EnableJpaAuditing` is configured

## Integration Patterns

### SAP Business One Integration
```java
@Service
public class SapIntegrationService {
    
    @Value("${sap.service-layer.url}")
    private String sapUrl;
    
    public void syncToSap(EntityDTO dto) {
        // RestTemplate or WebClient to call SAP Service Layer
        // Handle authentication with session cookies
        // Map JHipster entity to SAP format
    }
}
```

### Multi-tenant Support
```java
// Use discriminator columns or separate schemas
@Entity
@Table(name = "entity")
@Where(clause = "branch_id = :branchId")
public class Entity extends AbstractAuditingEntity {
    @Column(name = "branch_id")
    private Long branchId;
}
```

## When to Use This Skill
- Creating new JHipster entities and CRUD operations
- Implementing business logic in service layer
- Designing REST APIs following JHipster conventions
- Database schema evolution with Liquibase
- Security configuration for endpoints
- Integration with external systems (SAP, payment gateways)
- Performance optimization in JHipster apps
- Multi-branch/multi-tenant implementations

## Related Technologies
- Spring Boot 3.x
- Spring Data JPA
- Liquibase
- MapStruct
- JWT Authentication
- Hibernate
- PostgreSQL/MySQL/SQL Server
