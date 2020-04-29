```python
class QuerysetIsNotSortedByPkException(APIException):
    status_code = status.HTTP_400_BAD_REQUEST
    default_code = 'pk_pagination_error'
    default_detail = '쿼리결과가 pk기준으로 정렬되어 있지 않습니다'


class PkPagination(BasePagination):
    page_size_query_param = 'per_page'
    page_size = 50
    max_page_size = 1000

    def paginate_queryset(self, queryset, request, view=None):
        page_size = self.get_page_size(request)
        self.pk_field = queryset.model._meta.pk.attname
        order_by_options = (
                queryset.query.order_by or
                queryset.model._meta.ordering or
                (f'{self.pk_field}',)
        )
        order_by_reverse = tuple(
            order.replace('-', '')
            if order.startswith('-')
            else f'-{order}'
            for order in order_by_options
        )
        if f'{self.pk_field}' in order_by_options or 'pk' in order_by_options:
            relation_before, relation_after = 'lt', 'gt'
        elif f'-{self.pk_field}' in order_by_options or '-pk' in order_by_options:
            relation_before, relation_after = 'gt', 'lt'
        else:
            raise QuerysetIsNotSortedByPkException()

        lookup_before = f'{self.pk_field}__{relation_before}'
        lookup_after = f'{self.pk_field}__{relation_after}'

        target_pk = request.query_params.get('page_target_pk') or request.query_params.get(
            f'page_target_{self.pk_field}')
        prev_pk = request.query_params.get('page_prev_pk') or request.query_params.get(f'page_prev_{self.pk_field}')
        next_pk = request.query_params.get('page_next_pk') or request.query_params.get(f'page_next_{self.pk_field}')
        if target_pk:
            qs_before = queryset.order_by(*order_by_reverse).filter(**{f'{lookup_before}e': target_pk})[:page_size // 2]
            qs_after = queryset.filter(**{f'{lookup_after}': target_pk})[:page_size // 2]
            return qs_before.union(qs_after).order_by(*order_by_options)
        elif prev_pk:
            qs = queryset.order_by(*order_by_reverse).filter(**{lookup_before: prev_pk})[:page_size]
            return reversed(qs)
        elif next_pk:
            return queryset.filter(**{lookup_after: next_pk})[:page_size]
        return queryset[:page_size]

    def get_page_size(self, request):
        if self.page_size_query_param:
            try:
                return _positive_int(
                    request.query_params[self.page_size_query_param],
                    strict=True, cutoff=self.max_page_size)
            except (KeyError, ValueError):
                pass
        return self.page_size

    def get_paginated_response(self, data):
        return Response(OrderedDict([
            ('prev', {
                'name': f'page_prev_{self.pk_field}',
                'value': data[0][self.pk_field],
            }),
            ('next', {
                'name': f'page_next_{self.pk_field}',
                'value': data[-1][self.pk_field],
            }),
            ('results', data),
        ]))

    def get_paginated_response_schema(self, schema):
        return {
            'type': 'object',
            'properties': {
                'prev': {
                    'type': 'object',
                    'properties': {
                        'name': {
                            'type': 'string',
                        },
                        'value': {
                            'type': 'integer',
                        },
                    },
                },
                'next': {
                    'type': 'object',
                    'properties': {
                        'name': {
                            'type': 'string',
                        },
                        'value': {
                            'type': 'integer',
                        },
                    },
                },
                'results': schema,
            }
        }

    def get_schema_fields(self, view):
        assert coreapi is not None, 'coreapi must be installed to use `get_schema_fields()`'
        assert coreschema is not None, 'coreschema must be installed to use `get_schema_fields()`'
        fields = [
            coreapi.Field(
                name=f'page_target_pk',
                required=False,
                location='query',
                schema=coreschema.Integer(
                    title='Page target pk',
                    description='해당하는 row를 중심으로 페이징결과를 반환할 pk'
                )
            ),
            coreapi.Field(
                name=f'page_prev_pk',
                required=False,
                location='query',
                schema=coreschema.Integer(
                    title='Page prev pk',
                    description='주어진 row보다 이전의 페이징 결과를 반환할 pk'
                )
            ),
            coreapi.Field(
                name=f'page_next_pk',
                required=False,
                location='query',
                schema=coreschema.Integer(
                    title='Page next pk',
                    description='주어진 row 다음의 페이징 결과를 반환할 pk'
                )
            ),
        ]
        return fields

    def get_schema_operation_parameters(self, view):
        parameters = [
            {
                'name': 'page_target_pk',
                'required': False,
                'in': 'query',
                'description': 'schema_operation_parameter_target_pk',
                'schema': {
                    'type': 'integer',
                },
            },
            {
                'name': 'page_prev_pk',
                'required': False,
                'in': 'query',
                'description': 'schema_operation_parameter_prev_pk',
                'schema': {
                    'type': 'integer',
                },
            },
            {
                'name': 'page_next_pk',
                'required': False,
                'in': 'query',
                'description': 'schema_operation_parameter_next_pk',
                'schema': {
                    'type': 'integer',
                },
            },
        ]
        return parameters
```