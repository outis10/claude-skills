# JHipster React Component Examples

## List View Component
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
    dispatch(getEntities({
      page: paginationState.activePage - 1,
      size: paginationState.itemsPerPage,
      sort: `${paginationState.sort},${paginationState.order}`,
    }));
  };

  const sortEntities = () => {
    getAllEntities();
    const endURL = `?page=${paginationState.activePage}&sort=${paginationState.sort},${paginationState.order}`;
    if (location.search !== endURL) {
      navigate(`${location.pathname}${endURL}`);
    }
  };

  useEffect(() => { sortEntities(); },
    [paginationState.activePage, paginationState.order, paginationState.sort]);

  useEffect(() => {
    const params = new URLSearchParams(location.search);
    const page = params.get('page');
    const sort = params.get(SORT);
    if (page && sort) {
      const sortSplit = sort.split(',');
      setPaginationState({ ...paginationState, activePage: +page, sort: sortSplit[0], order: sortSplit[1] });
    }
  }, [location.search]);

  const sort = p => () =>
    setPaginationState({ ...paginationState, order: paginationState.order === ASC ? DESC : ASC, sort: p });

  return (
    <div>
      <h2 id="entity-heading" data-cy="EntityHeading">
        <Translate contentKey="myApp.entity.home.title">Entities</Translate>
        <div className="d-flex justify-content-end">
          <Button className="me-2" color="info" onClick={() => sortEntities()} disabled={loading}>
            <FontAwesomeIcon icon="sync" spin={loading} /> <Translate contentKey="myApp.entity.home.refreshListLabel">Refresh</Translate>
          </Button>
          <Link to="/entity/new" className="btn btn-primary jh-create-entity" data-cy="entityCreateButton">
            <FontAwesomeIcon icon="plus" /> <Translate contentKey="myApp.entity.home.createLabel">Create new Entity</Translate>
          </Link>
        </div>
      </h2>
      <div className="table-responsive">
        {entityList && entityList.length > 0 ? (
          <Table responsive>
            <thead>
              <tr>
                <th className="hand" onClick={sort('id')}>ID <FontAwesomeIcon icon="sort" /></th>
                <th className="hand" onClick={sort('name')}>Name <FontAwesomeIcon icon="sort" /></th>
                <th className="hand" onClick={sort('status')}>Status <FontAwesomeIcon icon="sort" /></th>
                <th />
              </tr>
            </thead>
            <tbody>
              {entityList.map((entity, i) => (
                <tr key={`entity-${i}`}>
                  <td><Button tag={Link} to={`/entity/${entity.id}`} color="link" size="sm">{entity.id}</Button></td>
                  <td>{entity.name}</td>
                  <td><Translate contentKey={`myApp.EntityStatus.${entity.status}`} /></td>
                  <td className="text-end">
                    <div className="btn-group flex-btn-group-container">
                      <Button tag={Link} to={`/entity/${entity.id}`} color="info" size="sm"><FontAwesomeIcon icon="eye" /></Button>
                      <Button tag={Link} to={`/entity/${entity.id}/edit`} color="primary" size="sm"><FontAwesomeIcon icon="pencil-alt" /></Button>
                      <Button tag={Link} to={`/entity/${entity.id}/delete`} color="danger" size="sm"><FontAwesomeIcon icon="trash" /></Button>
                    </div>
                  </td>
                </tr>
              ))}
            </tbody>
          </Table>
        ) : (!loading && <div className="alert alert-warning">No Entities found</div>)}
      </div>
      {totalItems ? (
        <div>
          <JhiItemCount page={paginationState.activePage} total={totalItems} itemsPerPage={paginationState.itemsPerPage} i18nEnabled />
          <JhiPagination activePage={paginationState.activePage} onSelect={p => setPaginationState({ ...paginationState, activePage: p })}
            maxButtons={5} itemsPerPage={paginationState.itemsPerPage} totalItems={totalItems} />
        </div>
      ) : ''}
    </div>
  );
};
export default Entity;
```

## Create/Update Form Component
```typescript
// src/main/webapp/app/entities/entity/entity-update.tsx
export const EntityUpdate = () => {
  const dispatch = useAppDispatch();
  const navigate = useNavigate();
  const { id } = useParams<'id'>();
  const isNew = id === undefined;

  const entityEntity = useAppSelector(state => state.entity.entity);
  const loading = useAppSelector(state => state.entity.loading);
  const updating = useAppSelector(state => state.entity.updating);
  const updateSuccess = useAppSelector(state => state.entity.updateSuccess);

  useEffect(() => { isNew ? dispatch(reset()) : dispatch(getEntity(id)); }, []);
  useEffect(() => { if (updateSuccess) navigate('/entity' + location.search); }, [updateSuccess]);

  const saveEntity = values => {
    const entity = { ...entityEntity, ...values };
    isNew ? dispatch(createEntity(entity)) : dispatch(updateEntity(entity));
  };

  return (
    <Row className="justify-content-center">
      <Col md="8">
        <h2><Translate contentKey="myApp.entity.home.createOrEditLabel">Create or edit Entity</Translate></h2>
        {loading ? <p>Loading...</p> : (
          <ValidatedForm defaultValues={isNew ? {} : { status: 'ACTIVE', ...entityEntity }} onSubmit={saveEntity}>
            {!isNew && <ValidatedField name="id" required readOnly label={translate('global.field.id')} />}
            <ValidatedField name="name" label={translate('myApp.entity.name')} type="text"
              validate={{ required: { value: true, message: translate('entity.validation.required') } }} />
            <ValidatedField name="description" label={translate('myApp.entity.description')} type="textarea" />
            <Button tag={Link} to="/entity" replace color="info"><FontAwesomeIcon icon="arrow-left" /> Back</Button>
            {' '}<Button color="primary" type="submit" disabled={updating}><FontAwesomeIcon icon="save" /> Save</Button>
          </ValidatedForm>
        )}
      </Col>
    </Row>
  );
};
```

## Detail View Component
```typescript
// src/main/webapp/app/entities/entity/entity-detail.tsx
export const EntityDetail = () => {
  const dispatch = useAppDispatch();
  const { id } = useParams<'id'>();

  useEffect(() => { dispatch(getEntity(id)); }, []);

  const entityEntity = useAppSelector(state => state.entity.entity);

  return (
    <Row>
      <Col md="8">
        <h2><Translate contentKey="myApp.entity.detail.title">Entity</Translate></h2>
        <dl className="jh-entity-details">
          <dt>ID</dt><dd>{entityEntity.id}</dd>
          <dt>Name</dt><dd>{entityEntity.name}</dd>
          <dt>Description</dt><dd>{entityEntity.description}</dd>
          <dt>Status</dt><dd><Translate contentKey={`myApp.EntityStatus.${entityEntity.status}`} /></dd>
          <dt>Created Date</dt>
          <dd>{entityEntity.createdDate ? <TextFormat value={entityEntity.createdDate} type="date" format={APP_DATE_FORMAT} /> : null}</dd>
        </dl>
        <Button tag={Link} to="/entity" replace color="info"><FontAwesomeIcon icon="arrow-left" /> Back</Button>
        {' '}<Button tag={Link} to={`/entity/${entityEntity.id}/edit`} replace color="primary"><FontAwesomeIcon icon="pencil-alt" /> Edit</Button>
      </Col>
    </Row>
  );
};
```

## WebSocket Integration
```typescript
import SockJS from 'sockjs-client';
import Stomp from 'webstomp-client';

let stompClient = null;

export const connectWebSocket = (store) => {
  const socket = new SockJS('/websocket/tracker');
  stompClient = Stomp.over(socket, { protocols: ['v12.stomp'] });
  stompClient.connect({}, () => {
    stompClient.subscribe('/topic/entity-updates', message => {
      store.dispatch({ type: 'entity/websocket_update', payload: JSON.parse(message.body) });
    });
  });
};
```

## Document Upload (Municipal Gazette)
```typescript
const DocumentUpload = () => {
  const [selectedFile, setSelectedFile] = useState<File | null>(null);
  const [preview, setPreview] = useState('');

  const handleFileChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (file) {
      setSelectedFile(file);
      const reader = new FileReader();
      reader.onloadend = () => setPreview(reader.result as string);
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

## Approval Workflow (Treasury)
```typescript
const ApprovalWorkflow = ({ transaction }) => {
  const currentUser = useAppSelector(state => state.authentication.account);

  const canApprove = useMemo(() =>
    transaction.currentApprovalLevel === currentUser.approvalLevel &&
    transaction.status === 'PENDING_APPROVAL',
    [transaction, currentUser]);

  return (
    <div className="approval-workflow">
      <h4>Approval Status: Level {transaction.currentApprovalLevel} of {transaction.requiredLevels}</h4>
      {canApprove && (
        <div>
          <Button color="success" onClick={() => dispatch(approveTransaction({ id: transaction.id, approved: true }))}>
            <FontAwesomeIcon icon="check" /> Approve
          </Button>
          <Button color="danger" onClick={() => dispatch(approveTransaction({ id: transaction.id, approved: false }))}>
            <FontAwesomeIcon icon="times" /> Reject
          </Button>
        </div>
      )}
      <Timeline approvalHistory={transaction.approvalHistory} />
    </div>
  );
};
```

## Testing
```typescript
import { render, screen } from '@testing-library/react';
import { Provider } from 'react-redux';
import { MemoryRouter } from 'react-router-dom';
import configureStore from 'redux-mock-store';

const mockStore = configureStore([]);

describe('Entity Component', () => {
  it('should render entity list', () => {
    const store = mockStore({ entity: { entities: [], loading: false, totalItems: 0 } });
    render(
      <Provider store={store}><MemoryRouter><Entity /></MemoryRouter></Provider>
    );
    expect(screen.getByText(/Entities/i)).toBeInTheDocument();
  });
});
```
