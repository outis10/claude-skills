---
name: jhipster-react-frontend
description: "JHipster React frontend development patterns. Use when working with: JHipster React components, Redux Toolkit state management, entity CRUD views, React Router configuration, form validation with ValidatedForm/ValidatedField, Axios API integration, TypeScript entity models, Reactstrap/Bootstrap layouts, i18n, or WebSocket real-time updates. Triggers: 'genera componente react', 'crea vista de entidad', 'agrega formulario', React TypeScript components in JHipster projects."
---

# JHipster React Frontend Development

## Project Structure (JHipster Convention)
```
src/main/webapp/app/
├── config/               # store.ts, constants.ts
├── entities/             # Entity modules (list, detail, update, delete, reducer, routes)
├── modules/              # Feature modules (administration, account)
├── shared/               # Shared components, models, reducers, utils
└── app.tsx               # Root component
```

## TypeScript Entity Model
```typescript
export interface IEntity {
  id?: number;
  name?: string;
  description?: string | null;
  amount?: number;
  status?: EntityStatus;
  createdDate?: string;
  createdBy?: string;
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

## Redux Slice Pattern
```typescript
import { createAsyncThunk, createSlice } from '@reduxjs/toolkit';
import axios from 'axios';
import { serializeAxiosError } from 'app/shared/reducers/reducer.utils';

const apiUrl = 'api/entities';

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
  async (id: string | number) => axios.get<IEntity>(`${apiUrl}/${id}`),
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
    const result = await axios.delete<IEntity>(`${apiUrl}/${id}`);
    thunkAPI.dispatch(getEntities({}));
    return result;
  },
  { serializeError: serializeAxiosError }
);

export const EntitySlice = createSlice({
  name: 'entity',
  initialState: {
    loading: false,
    errorMessage: null as string | null,
    entities: [] as IEntity[],
    entity: {} as IEntity,
    updating: false,
    updateSuccess: false,
    totalItems: 0,
  },
  reducers: {
    reset() { return EntitySlice.getInitialState(); },
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
        state.updating = false; state.updateSuccess = true;
        state.entity = action.payload.data;
      })
      .addCase(updateEntity.fulfilled, (state, action) => {
        state.updating = false; state.updateSuccess = true;
        state.entity = action.payload.data;
      })
      .addCase(deleteEntity.fulfilled, state => {
        state.updating = false; state.updateSuccess = true; state.entity = {};
      })
      .addMatcher(action => action.type.endsWith('/pending'), state => {
        state.errorMessage = null; state.updateSuccess = false;
        state.loading = true; state.updating = true;
      })
      .addMatcher(action => action.type.endsWith('/rejected'), (state, action) => {
        state.loading = false; state.updating = false;
        state.updateSuccess = false; state.errorMessage = action.error.message;
      });
  },
});

export const { reset } = EntitySlice.actions;
export default EntitySlice.reducer;
```

## Routes Configuration
```typescript
import { Route } from 'react-router-dom';
import ErrorBoundaryRoutes from 'app/shared/error/error-boundary-routes';

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
```

## Form Validation Pattern
```typescript
<ValidatedForm defaultValues={defaultValues()} onSubmit={saveEntity}>
  <ValidatedField name="name" label={translate('myApp.entity.name')} type="text"
    validate={{
      required: { value: true, message: translate('entity.validation.required') },
      maxLength: { value: 255, message: translate('entity.validation.maxlength', { max: 255 }) },
    }}
  />
  <ValidatedField name="amount" label={translate('myApp.entity.amount')} type="text"
    validate={{
      required: { value: true, message: translate('entity.validation.required') },
      validate: v => isNumber(v) || translate('entity.validation.number'),
    }}
  />
  <ValidatedField name="status" label={translate('myApp.entity.status')} type="select">
    {Object.keys(EntityStatus).map(s => (
      <option value={s} key={s}>{translate('myApp.EntityStatus.' + s)}</option>
    ))}
  </ValidatedField>
  <Button type="submit" color="primary" disabled={updating}>Save</Button>
</ValidatedForm>
```

## Custom Filter Hook
```typescript
export const useEntityFilter = (getEntitiesAction) => {
  const dispatch = useAppDispatch();
  const [filters, setFilters] = useState({ search: '', status: '', dateFrom: '', dateTo: '' });

  useEffect(() => {
    const delay = setTimeout(() => dispatch(getEntitiesAction(filters)), 300);
    return () => clearTimeout(delay);
  }, [filters]);

  return { filters, setFilters };
};
```

## i18n Structure
```json
{
  "myApp": {
    "entity": {
      "home": { "title": "Entities", "createLabel": "Create new Entity", "notFound": "No Entities found" },
      "name": "Name", "description": "Description", "amount": "Amount", "status": "Status"
    },
    "EntityStatus": { "ACTIVE": "Active", "INACTIVE": "Inactive", "PENDING": "Pending" }
  }
}
```

## Best Practices
- Use Redux Toolkit (`createSlice`, `createAsyncThunk`) with typed hooks (`useAppSelector`, `useAppDispatch`)
- Define TypeScript interfaces in `shared/model/`, avoid `any`
- Use `ValidatedForm`/`ValidatedField` from react-jhipster for forms
- Implement pagination for all list views
- Debounce search inputs (300ms)
- Lazy load routes with `React.lazy`
- Use `React.memo` and `useCallback` for performance
- Disable submit buttons during API calls (`disabled={updating}`)
- Handle loading, error, and success states in every view
- Translate all UI strings with i18n keys

## Integration with Backend
- Axios interceptors handle JWT automatically via localStorage
- API endpoints follow REST: `/api/entities`
- HTTP methods: GET (read), POST (create), PUT (update), DELETE (delete)
- Error responses: use `response.data.message`

For detailed component examples (List, Detail, Update, Delete, WebSocket, Approval Workflow, Document Upload), see `references/component-examples.md`.

## Related Technologies
React 18, TypeScript 5, Redux Toolkit, React Router 6, Reactstrap (Bootstrap 5), FontAwesome, react-jhipster, SockJS + STOMP, Axios
