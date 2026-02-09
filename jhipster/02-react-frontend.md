# JHipster React Frontend Development Skill

## Description
Use this skill when working with JHipster React frontend: creating components, Redux state management, entity CRUD views, routing, form validation with React Hook Form, API integration with Axios, TypeScript interfaces, responsive layouts with Reactstrap/Bootstrap, and authentication flows. Triggers include: mentions of JHipster React, TypeScript React components, Redux Toolkit, entity management UI, form validation, or React Router configuration.

## Core Patterns

### 1. Project Structure (JHipster React Convention)
```
src/main/webapp/app/
├── config/                    # App configuration
│   ├── store.ts              # Redux store setup
│   └── constants.ts          # API endpoints, app constants
├── entities/                  # Entity modules
│   └── entity-name/
│       ├── entity-name.tsx           # List view
│       ├── entity-name-detail.tsx    # Detail view
│       ├── entity-name-update.tsx    # Create/Edit form
│       ├── entity-name-delete-dialog.tsx
│       ├── entity-name.reducer.ts    # Redux slice
│       └── index.tsx                 # Route configuration
├── modules/                   # Feature modules
│   ├── administration/
│   └── account/
├── shared/                    # Shared components
│   ├── layout/
│   ├── model/                # TypeScript interfaces
│   ├── reducers/             # Root reducer
│   └── util/                 # Utility functions
└── app.tsx                   # Root component
```

### 2. TypeScript Entity Model
```typescript
// src/main/webapp/app/shared/model/entity.model.ts
export interface IEntity {
  id?: number;
  name?: string;
  description?: string | null;
  amount?: number;
  status?: EntityStatus;
  createdDate?: string;
  createdBy?: string;
  lastModifiedDate?: string;
  lastModifiedBy?: string;
  // Relationships
  relatedEntity?: IRelatedEntity | null;
  relatedEntityId?: number | null;
}

export enum EntityStatus {
  ACTIVE = 'ACTIVE',
  INACTIVE = 'INACTIVE',
  PENDING = 'PENDING',
}

export const defaultValue: Readonly<IEntity> = {
  status: EntityStatus.ACTIVE,
};
```

### 3. Redux Slice with Redux Toolkit
```typescript
// src/main/webapp/app/entities/entity/entity.reducer.ts
import { createAsyncThunk, createSlice } from '@reduxjs/toolkit';
import axios from 'axios';
import { IEntity } from 'app/shared/model/entity.model';
import { serializeAxiosError } from 'app/shared/reducers/reducer.utils';

const apiUrl = 'api/entities';

interface EntityState {
  loading: boolean;
  errorMessage: string | null;
  entities: IEntity[];
  entity: IEntity;
  updating: boolean;
  updateSuccess: boolean;
  totalItems: number;
}

const initialState: EntityState = {
  loading: false,
  errorMessage: null,
  entities: [],
  entity: {},
  updating: false,
  updateSuccess: false,
  totalItems: 0,
};

// Async thunks
export const getEntities = createAsyncThunk(
  'entity/fetch_entity_list',
  async ({ page, size, sort }: { page?: number; size?: number; sort?: string }) => {
    const requestUrl = `${apiUrl}${sort ? `?page=${page}&size=${size}&sort=${sort}` : ''}`;
    return axios.get<IEntity[]>(requestUrl);
  },
  { serializeError: serializeAxiosError }
);

export const getEntity = createAsyncThunk(
  'entity/fetch_entity',
  async (id: string | number) => {
    const requestUrl = `${apiUrl}/${id}`;
    return axios.get<IEntity>(requestUrl);
  },
  { serializeError: serializeAxiosError }
);

export const createEntity = createAsyncThunk(
  'entity/create_entity',
  async (entity: IEntity, thunkAPI) => {
    const result = await axios.post<IEntity>(apiUrl, entity);
    thunkAPI.dispatch(getEntities({}));
    return result;
  },
  { serializeError: serializeAxiosError }
);

export const updateEntity = createAsyncThunk(
  'entity/update_entity',
  async (entity: IEntity, thunkAPI) => {
    const result = await axios.put<IEntity>(`${apiUrl}/${entity.id}`, entity);
    thunkAPI.dispatch(getEntities({}));
    return result;
  },
  { serializeError: serializeAxiosError }
);

export const deleteEntity = createAsyncThunk(
  'entity/delete_entity',
  async (id: string | number, thunkAPI) => {
    const requestUrl = `${apiUrl}/${id}`;
    const result = await axios.delete<IEntity>(requestUrl);
    thunkAPI.dispatch(getEntities({}));
    return result;
  },
  { serializeError: serializeAxiosError }
);

// Slice
export const EntitySlice = createSlice({
  name: 'entity',
  initialState,
  reducers: {
    reset() {
      return initialState;
    },
  },
  extraReducers(builder) {
    builder
      .addCase(getEntity.fulfilled, (state, action) => {
        state.loading = false;
        state.entity = action.payload.data;
      })
      .addCase(getEntities.fulfilled, (state, action) => {
        state.loading = false;
        state.entities = action.payload.data;
        state.totalItems = parseInt(action.payload.headers['x-total-count'], 10);
      })
      .addCase(createEntity.fulfilled, (state, action) => {
        state.updating = false;
        state.updateSuccess = true;
        state.entity = action.payload.data;
      })
      .addCase(updateEntity.fulfilled, (state, action) => {
        state.updating = false;
        state.updateSuccess = true;
        state.entity = action.payload.data;
      })
      .addCase(deleteEntity.fulfilled, state => {
        state.updating = false;
        state.updateSuccess = true;
        state.entity = {};
      })
      .addMatcher(
        action => action.type.endsWith('/pending'),
        state => {
          state.errorMessage = null;
          state.updateSuccess = false;
          state.loading = true;
          state.updating = true;
        }
      )
      .addMatcher(
        action => action.type.endsWith('/rejected'),
        (state, action) => {
          state.loading = false;
          state.updating = false;
          state.updateSuccess = false;
          state.errorMessage = action.error.message;
        }
      );
  },
});

export const { reset } = EntitySlice.actions;
export default EntitySlice.reducer;
```

### 4. List View Component
```typescript
// src/main/webapp/app/entities/entity/entity.tsx
import React, { useEffect, useState } from 'react';
import { Link, useLocation, useNavigate } from 'react-router-dom';
import { Button, Table } from 'reactstrap';
import { Translate, getSortState, JhiPagination, JhiItemCount } from 'react-jhipster';
import { FontAwesomeIcon } from '@fortawesome/react-fontawesome';
import { useAppDispatch, useAppSelector } from 'app/config/store';
import { getEntities } from './entity.reducer';
import { ASC, DESC, ITEMS_PER_PAGE, SORT } from 'app/shared/util/pagination.constants';
import { overridePaginationStateWithQueryParams } from 'app/shared/util/entity-utils';

export const Entity = () => {
  const dispatch = useAppDispatch();
  const location = useLocation();
  const navigate = useNavigate();

  const [paginationState, setPaginationState] = useState(
    overridePaginationStateWithQueryParams(getSortState(location, ITEMS_PER_PAGE, 'id'), location.search)
  );

  const entityList = useAppSelector(state => state.entity.entities);
  const loading = useAppSelector(state => state.entity.loading);
  const totalItems = useAppSelector(state => state.entity.totalItems);

  const getAllEntities = () => {
    dispatch(
      getEntities({
        page: paginationState.activePage - 1,
        size: paginationState.itemsPerPage,
        sort: `${paginationState.sort},${paginationState.order}`,
      })
    );
  };

  const sortEntities = () => {
    getAllEntities();
    const endURL = `?page=${paginationState.activePage}&sort=${paginationState.sort},${paginationState.order}`;
    if (location.search !== endURL) {
      navigate(`${location.pathname}${endURL}`);
    }
  };

  useEffect(() => {
    sortEntities();
  }, [paginationState.activePage, paginationState.order, paginationState.sort]);

  useEffect(() => {
    const params = new URLSearchParams(location.search);
    const page = params.get('page');
    const sort = params.get(SORT);
    if (page && sort) {
      const sortSplit = sort.split(',');
      setPaginationState({
        ...paginationState,
        activePage: +page,
        sort: sortSplit[0],
        order: sortSplit[1],
      });
    }
  }, [location.search]);

  const sort = p => () => {
    setPaginationState({
      ...paginationState,
      order: paginationState.order === ASC ? DESC : ASC,
      sort: p,
    });
  };

  const handlePagination = currentPage =>
    setPaginationState({
      ...paginationState,
      activePage: currentPage,
    });

  const handleSyncList = () => {
    sortEntities();
  };

  return (
    <div>
      <h2 id="entity-heading" data-cy="EntityHeading">
        <Translate contentKey="myApp.entity.home.title">Entities</Translate>
        <div className="d-flex justify-content-end">
          <Button className="me-2" color="info" onClick={handleSyncList} disabled={loading}>
            <FontAwesomeIcon icon="sync" spin={loading} />{' '}
            <Translate contentKey="myApp.entity.home.refreshListLabel">Refresh List</Translate>
          </Button>
          <Link to="/entity/new" className="btn btn-primary jh-create-entity" id="jh-create-entity" data-cy="entityCreateButton">
            <FontAwesomeIcon icon="plus" />
            &nbsp;
            <Translate contentKey="myApp.entity.home.createLabel">Create new Entity</Translate>
          </Link>
        </div>
      </h2>
      <div className="table-responsive">
        {entityList && entityList.length > 0 ? (
          <Table responsive>
            <thead>
              <tr>
                <th className="hand" onClick={sort('id')}>
                  <Translate contentKey="myApp.entity.id">ID</Translate> <FontAwesomeIcon icon="sort" />
                </th>
                <th className="hand" onClick={sort('name')}>
                  <Translate contentKey="myApp.entity.name">Name</Translate> <FontAwesomeIcon icon="sort" />
                </th>
                <th className="hand" onClick={sort('status')}>
                  <Translate contentKey="myApp.entity.status">Status</Translate> <FontAwesomeIcon icon="sort" />
                </th>
                <th />
              </tr>
            </thead>
            <tbody>
              {entityList.map((entity, i) => (
                <tr key={`entity-${i}`} data-cy="entityDetailsButton">
                  <td>
                    <Button tag={Link} to={`/entity/${entity.id}`} color="link" size="sm">
                      {entity.id}
                    </Button>
                  </td>
                  <td>{entity.name}</td>
                  <td>
                    <Translate contentKey={`myApp.EntityStatus.${entity.status}`} />
                  </td>
                  <td className="text-end">
                    <div className="btn-group flex-btn-group-container">
                      <Button tag={Link} to={`/entity/${entity.id}`} color="info" size="sm" data-cy="entityDetailsButton">
                        <FontAwesomeIcon icon="eye" />{' '}
                        <span className="d-none d-md-inline">
                          <Translate contentKey="entity.action.view">View</Translate>
                        </span>
                      </Button>
                      <Button
                        tag={Link}
                        to={`/entity/${entity.id}/edit?page=${paginationState.activePage}&sort=${paginationState.sort},${paginationState.order}`}
                        color="primary"
                        size="sm"
                        data-cy="entityEditButton"
                      >
                        <FontAwesomeIcon icon="pencil-alt" />{' '}
                        <span className="d-none d-md-inline">
                          <Translate contentKey="entity.action.edit">Edit</Translate>
                        </span>
                      </Button>
                      <Button
                        tag={Link}
                        to={`/entity/${entity.id}/delete?page=${paginationState.activePage}&sort=${paginationState.sort},${paginationState.order}`}
                        color="danger"
                        size="sm"
                        data-cy="entityDeleteButton"
                      >
                        <FontAwesomeIcon icon="trash" />{' '}
                        <span className="d-none d-md-inline">
                          <Translate contentKey="entity.action.delete">Delete</Translate>
                        </span>
                      </Button>
                    </div>
                  </td>
                </tr>
              ))}
            </tbody>
          </Table>
        ) : (
          !loading && (
            <div className="alert alert-warning">
              <Translate contentKey="myApp.entity.home.notFound">No Entities found</Translate>
            </div>
          )
        )}
      </div>
      {totalItems ? (
        <div className={entityList && entityList.length > 0 ? '' : 'd-none'}>
          <div className="justify-content-center d-flex">
            <JhiItemCount page={paginationState.activePage} total={totalItems} itemsPerPage={paginationState.itemsPerPage} i18nEnabled />
          </div>
          <div className="justify-content-center d-flex">
            <JhiPagination
              activePage={paginationState.activePage}
              onSelect={handlePagination}
              maxButtons={5}
              itemsPerPage={paginationState.itemsPerPage}
              totalItems={totalItems}
            />
          </div>
        </div>
      ) : (
        ''
      )}
    </div>
  );
};

export default Entity;
```

### 5. Create/Update Form Component
```typescript
// src/main/webapp/app/entities/entity/entity-update.tsx
import React, { useEffect } from 'react';
import { Link, useNavigate, useParams } from 'react-router-dom';
import { Button, Row, Col, FormText } from 'reactstrap';
import { isNumber, Translate, translate, ValidatedField, ValidatedForm } from 'react-jhipster';
import { FontAwesomeIcon } from '@fortawesome/react-fontawesome';
import { useAppDispatch, useAppSelector } from 'app/config/store';
import { getEntity, updateEntity, createEntity, reset } from './entity.reducer';
import { EntityStatus } from 'app/shared/model/entity.model';

export const EntityUpdate = () => {
  const dispatch = useAppDispatch();
  const navigate = useNavigate();
  const { id } = useParams<'id'>();
  const isNew = id === undefined;

  const entityEntity = useAppSelector(state => state.entity.entity);
  const loading = useAppSelector(state => state.entity.loading);
  const updating = useAppSelector(state => state.entity.updating);
  const updateSuccess = useAppSelector(state => state.entity.updateSuccess);

  const entityStatusValues = Object.keys(EntityStatus);

  const handleClose = () => {
    navigate('/entity' + location.search);
  };

  useEffect(() => {
    if (isNew) {
      dispatch(reset());
    } else {
      dispatch(getEntity(id));
    }
  }, []);

  useEffect(() => {
    if (updateSuccess) {
      handleClose();
    }
  }, [updateSuccess]);

  const saveEntity = values => {
    const entity = {
      ...entityEntity,
      ...values,
    };

    if (isNew) {
      dispatch(createEntity(entity));
    } else {
      dispatch(updateEntity(entity));
    }
  };

  const defaultValues = () =>
    isNew
      ? {}
      : {
          status: 'ACTIVE',
          ...entityEntity,
        };

  return (
    <div>
      <Row className="justify-content-center">
        <Col md="8">
          <h2 id="myApp.entity.home.createOrEditLabel" data-cy="EntityCreateUpdateHeading">
            <Translate contentKey="myApp.entity.home.createOrEditLabel">Create or edit a Entity</Translate>
          </h2>
        </Col>
      </Row>
      <Row className="justify-content-center">
        <Col md="8">
          {loading ? (
            <p>Loading...</p>
          ) : (
            <ValidatedForm defaultValues={defaultValues()} onSubmit={saveEntity}>
              {!isNew ? (
                <ValidatedField
                  name="id"
                  required
                  readOnly
                  id="entity-id"
                  label={translate('global.field.id')}
                  validate={{ required: true }}
                />
              ) : null}
              <ValidatedField
                label={translate('myApp.entity.name')}
                id="entity-name"
                name="name"
                data-cy="name"
                type="text"
                validate={{
                  required: { value: true, message: translate('entity.validation.required') },
                  maxLength: { value: 255, message: translate('entity.validation.maxlength', { max: 255 }) },
                }}
              />
              <ValidatedField
                label={translate('myApp.entity.description')}
                id="entity-description"
                name="description"
                data-cy="description"
                type="textarea"
              />
              <ValidatedField
                label={translate('myApp.entity.amount')}
                id="entity-amount"
                name="amount"
                data-cy="amount"
                type="text"
                validate={{
                  required: { value: true, message: translate('entity.validation.required') },
                  validate: v => isNumber(v) || translate('entity.validation.number'),
                }}
              />
              <ValidatedField
                label={translate('myApp.entity.status')}
                id="entity-status"
                name="status"
                data-cy="status"
                type="select"
              >
                {entityStatusValues.map(status => (
                  <option value={status} key={status}>
                    {translate('myApp.EntityStatus.' + status)}
                  </option>
                ))}
              </ValidatedField>
              <Button tag={Link} id="cancel-save" data-cy="entityCreateCancelButton" to="/entity" replace color="info">
                <FontAwesomeIcon icon="arrow-left" />
                &nbsp;
                <span className="d-none d-md-inline">
                  <Translate contentKey="entity.action.back">Back</Translate>
                </span>
              </Button>
              &nbsp;
              <Button color="primary" id="save-entity" data-cy="entityCreateSaveButton" type="submit" disabled={updating}>
                <FontAwesomeIcon icon="save" />
                &nbsp;
                <Translate contentKey="entity.action.save">Save</Translate>
              </Button>
            </ValidatedForm>
          )}
        </Col>
      </Row>
    </div>
  );
};

export default EntityUpdate;
```

### 6. Detail View Component
```typescript
// src/main/webapp/app/entities/entity/entity-detail.tsx
import React, { useEffect } from 'react';
import { Link, useParams } from 'react-router-dom';
import { Button, Row, Col } from 'reactstrap';
import { Translate, TextFormat } from 'react-jhipster';
import { FontAwesomeIcon } from '@fortawesome/react-fontawesome';
import { APP_DATE_FORMAT } from 'app/config/constants';
import { useAppDispatch, useAppSelector } from 'app/config/store';
import { getEntity } from './entity.reducer';

export const EntityDetail = () => {
  const dispatch = useAppDispatch();
  const { id } = useParams<'id'>();

  useEffect(() => {
    dispatch(getEntity(id));
  }, []);

  const entityEntity = useAppSelector(state => state.entity.entity);

  return (
    <Row>
      <Col md="8">
        <h2 data-cy="entityDetailsHeading">
          <Translate contentKey="myApp.entity.detail.title">Entity</Translate>
        </h2>
        <dl className="jh-entity-details">
          <dt>
            <span id="id">
              <Translate contentKey="global.field.id">ID</Translate>
            </span>
          </dt>
          <dd>{entityEntity.id}</dd>
          <dt>
            <span id="name">
              <Translate contentKey="myApp.entity.name">Name</Translate>
            </span>
          </dt>
          <dd>{entityEntity.name}</dd>
          <dt>
            <span id="description">
              <Translate contentKey="myApp.entity.description">Description</Translate>
            </span>
          </dt>
          <dd>{entityEntity.description}</dd>
          <dt>
            <span id="amount">
              <Translate contentKey="myApp.entity.amount">Amount</Translate>
            </span>
          </dt>
          <dd>{entityEntity.amount}</dd>
          <dt>
            <span id="status">
              <Translate contentKey="myApp.entity.status">Status</Translate>
            </span>
          </dt>
          <dd>
            <Translate contentKey={`myApp.EntityStatus.${entityEntity.status}`} />
          </dd>
          <dt>
            <span id="createdDate">
              <Translate contentKey="myApp.entity.createdDate">Created Date</Translate>
            </span>
          </dt>
          <dd>
            {entityEntity.createdDate ? (
              <TextFormat value={entityEntity.createdDate} type="date" format={APP_DATE_FORMAT} />
            ) : null}
          </dd>
        </dl>
        <Button tag={Link} to="/entity" replace color="info" data-cy="entityDetailsBackButton">
          <FontAwesomeIcon icon="arrow-left" />{' '}
          <span className="d-none d-md-inline">
            <Translate contentKey="entity.action.back">Back</Translate>
          </span>
        </Button>
        &nbsp;
        <Button tag={Link} to={`/entity/${entityEntity.id}/edit`} replace color="primary">
          <FontAwesomeIcon icon="pencil-alt" />{' '}
          <span className="d-none d-md-inline">
            <Translate contentKey="entity.action.edit">Edit</Translate>
          </span>
        </Button>
      </Col>
    </Row>
  );
};

export default EntityDetail;
```

### 7. Routes Configuration
```typescript
// src/main/webapp/app/entities/entity/index.tsx
import React from 'react';
import { Route } from 'react-router-dom';
import ErrorBoundaryRoutes from 'app/shared/error/error-boundary-routes';
import Entity from './entity';
import EntityDetail from './entity-detail';
import EntityUpdate from './entity-update';
import EntityDeleteDialog from './entity-delete-dialog';

const EntityRoutes = () => (
  <ErrorBoundaryRoutes>
    <Route index element={<Entity />} />
    <Route path="new" element={<EntityUpdate />} />
    <Route path=":id">
      <Route index element={<EntityDetail />} />
      <Route path="edit" element={<EntityUpdate />} />
      <Route path="delete" element={<EntityDeleteDialog />} />
    </Route>
  </ErrorBoundaryRoutes>
);

export default EntityRoutes;
```

### 8. Custom Hooks Pattern
```typescript
// src/main/webapp/app/shared/hooks/use-entity-filter.ts
import { useState, useEffect } from 'react';
import { useAppDispatch } from 'app/config/store';

export const useEntityFilter = (getEntitiesAction) => {
  const dispatch = useAppDispatch();
  const [filters, setFilters] = useState({
    search: '',
    status: '',
    dateFrom: '',
    dateTo: '',
  });

  useEffect(() => {
    const delayDebounceFn = setTimeout(() => {
      dispatch(getEntitiesAction(filters));
    }, 300);

    return () => clearTimeout(delayDebounceFn);
  }, [filters]);

  return { filters, setFilters };
};
```

### 9. Advanced Table with Filters
```typescript
// Component with search and filters
const EntityWithFilters = () => {
  const { filters, setFilters } = useEntityFilter(getEntities);

  return (
    <div>
      <Row className="mb-3">
        <Col md="4">
          <ValidatedField
            type="text"
            name="search"
            placeholder={translate('myApp.entity.home.search')}
            value={filters.search}
            onChange={e => setFilters({ ...filters, search: e.target.value })}
          />
        </Col>
        <Col md="3">
          <ValidatedField
            type="select"
            name="status"
            value={filters.status}
            onChange={e => setFilters({ ...filters, status: e.target.value })}
          >
            <option value="">All Statuses</option>
            {entityStatusValues.map(status => (
              <option value={status} key={status}>
                {translate(`myApp.EntityStatus.${status}`)}
              </option>
            ))}
          </ValidatedField>
        </Col>
      </Row>
      {/* Table here */}
    </div>
  );
};
```

### 10. WebSocket Integration (Real-time Updates)
```typescript
// src/main/webapp/app/config/websocket-middleware.ts
import SockJS from 'sockjs-client';
import Stomp from 'webstomp-client';
import { Store } from '@reduxjs/toolkit';

let stompClient = null;

export const connectWebSocket = (store: Store) => {
  const socket = new SockJS('/websocket/tracker');
  stompClient = Stomp.over(socket, { protocols: ['v12.stomp'] });

  stompClient.connect({}, () => {
    stompClient.subscribe('/topic/entity-updates', message => {
      const update = JSON.parse(message.body);
      // Dispatch action to update state
      store.dispatch({ type: 'entity/websocket_update', payload: update });
    });
  });
};

export const disconnectWebSocket = () => {
  if (stompClient !== null) {
    stompClient.disconnect();
  }
};
```

## Best Practices

### State Management
- ✅ Use Redux Toolkit (createSlice, createAsyncThunk)
- ✅ Keep component state minimal, lift shared state to Redux
- ✅ Use `useAppSelector` and `useAppDispatch` hooks (typed)
- ✅ Reset state on component unmount when needed
- ✅ Handle loading, error, and success states properly

### TypeScript
- ✅ Define interfaces for all entities in `shared/model/`
- ✅ Use strict typing for props and state
- ✅ Avoid `any` type, use `unknown` if necessary
- ✅ Export default values for entities
- ✅ Use enums for status fields

### Forms & Validation
- ✅ Use `ValidatedForm` and `ValidatedField` from react-jhipster
- ✅ Define validation rules inline
- ✅ Translate error messages with i18n keys
- ✅ Handle both client and server-side validation errors
- ✅ Disable submit button during API calls (`disabled={updating}`)

### Performance
- ✅ Use React.memo for expensive components
- ✅ Implement pagination for large lists
- ✅ Debounce search inputs (300ms)
- ✅ Lazy load routes with React.lazy
- ✅ Use useCallback for event handlers passed to children

### Internationalization (i18n)
```json
// src/main/webapp/i18n/en/entity.json
{
  "myApp": {
    "entity": {
      "home": {
        "title": "Entities",
        "createLabel": "Create new Entity",
        "refreshListLabel": "Refresh list",
        "notFound": "No Entities found",
        "search": "Search for Entity"
      },
      "detail": {
        "title": "Entity"
      },
      "name": "Name",
      "description": "Description",
      "amount": "Amount",
      "status": "Status"
    },
    "EntityStatus": {
      "ACTIVE": "Active",
      "INACTIVE": "Inactive",
      "PENDING": "Pending"
    }
  }
}
```

### Error Handling
```typescript
// Display API errors to user
import { toast } from 'react-toastify';

.addMatcher(
  action => action.type.endsWith('/rejected'),
  (state, action) => {
    state.loading = false;
    state.updating = false;
    toast.error(action.error.message || 'An error occurred');
  }
);
```

### Testing
```typescript
// entity.spec.tsx
import { render, screen } from '@testing-library/react';
import { Provider } from 'react-redux';
import { MemoryRouter } from 'react-router-dom';
import configureStore from 'redux-mock-store';
import Entity from './entity';

const mockStore = configureStore([]);

describe('Entity Component', () => {
  let store;

  beforeEach(() => {
    store = mockStore({
      entity: {
        entities: [],
        loading: false,
        totalItems: 0,
      },
    });
  });

  it('should render entity list', () => {
    render(
      <Provider store={store}>
        <MemoryRouter>
          <Entity />
        </MemoryRouter>
      </Provider>
    );
    
    expect(screen.getByText(/Entities/i)).toBeInTheDocument();
  });
});
```

## Common Patterns for Your Projects

### Municipal Gazette System (Document Management)
```typescript
// Document upload with preview
const DocumentUpload = () => {
  const [selectedFile, setSelectedFile] = useState<File | null>(null);
  const [preview, setPreview] = useState<string>('');

  const handleFileChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (file) {
      setSelectedFile(file);
      // Generate preview for PDFs
      const reader = new FileReader();
      reader.onloadend = () => {
        setPreview(reader.result as string);
      };
      reader.readAsDataURL(file);
    }
  };

  const handleUpload = async () => {
    if (!selectedFile) return;
    
    const formData = new FormData();
    formData.append('file', selectedFile);
    
    await dispatch(uploadDocument(formData));
  };

  return (
    <div>
      <input type="file" accept=".pdf" onChange={handleFileChange} />
      {preview && <embed src={preview} width="100%" height="600px" />}
      <Button onClick={handleUpload}>Upload</Button>
    </div>
  );
};
```

### Treasury System (Approval Workflow)
```typescript
// Multi-level approval component
const ApprovalWorkflow = ({ transaction }) => {
  const currentUser = useAppSelector(state => state.authentication.account);
  
  const canApprove = useMemo(() => {
    return transaction.currentApprovalLevel === currentUser.approvalLevel &&
           transaction.status === 'PENDING_APPROVAL';
  }, [transaction, currentUser]);

  const handleApprove = () => {
    dispatch(approveTransaction({ id: transaction.id, approved: true }));
  };

  const handleReject = () => {
    dispatch(approveTransaction({ id: transaction.id, approved: false }));
  };

  return (
    <div className="approval-workflow">
      <h4>Approval Status: Level {transaction.currentApprovalLevel} of {transaction.requiredLevels}</h4>
      {canApprove && (
        <div>
          <Button color="success" onClick={handleApprove}>
            <FontAwesomeIcon icon="check" /> Approve
          </Button>
          <Button color="danger" onClick={handleReject}>
            <FontAwesomeIcon icon="times" /> Reject
          </Button>
        </div>
      )}
      <Timeline approvalHistory={transaction.approvalHistory} />
    </div>
  );
};
```

### Real-time Notifications
```typescript
// Notification toast component
import { toast } from 'react-toastify';

useEffect(() => {
  if (stompClient) {
    stompClient.subscribe('/user/queue/notifications', message => {
      const notification = JSON.parse(message.body);
      
      toast.info(
        <div>
          <strong>{notification.title}</strong>
          <p>{notification.message}</p>
        </div>,
        {
          autoClose: 5000,
          position: 'top-right',
        }
      );
      
      // Update notification badge count
      dispatch(incrementNotificationCount());
    });
  }
}, [stompClient]);
```

## When to Use This Skill
- Creating new entity CRUD interfaces in JHipster React
- Implementing Redux state management for entities
- Building forms with validation
- Adding real-time features with WebSockets
- Implementing pagination, sorting, filtering
- Creating responsive layouts with Reactstrap
- Setting up routing for entity modules
- Internationalization of UI components
- Multi-level approval workflows
- Document upload and preview features

## Integration with Backend
- Always use axios interceptors configured by JHipster
- JWT tokens handled automatically via localStorage
- Error responses formatted by backend (use response.data.message)
- API endpoints follow REST conventions: `/api/entities`
- Use proper HTTP methods: GET (read), POST (create), PUT (update), DELETE (delete)

## Related Technologies
- React 18
- TypeScript 5
- Redux Toolkit
- React Router 6
- Reactstrap (Bootstrap 5)
- React Hook Form
- FontAwesome icons
- react-jhipster utilities
- SockJS + STOMP (WebSocket)
- Axios
