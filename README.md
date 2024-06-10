# pytest-crash
 test_post _________________________________________________________________________________________________________________ 

published_category = <Category: Even Special Coach Safe Because Ahead>, published_location = <Location: Mr. Theodore Neal>, user_client = <django.test.client.Client object at 0x0000018FC4E138E0>
another_user_client = <django.test.client.Client object at 0x0000018FC4E133A0>, unlogged_client = <django.test.client.Client object at 0x0000018FC4E13A00>, comment_to_a_post = <Comment: Comment object (6)>
create_post_context_form_item = KeyVal(key='form', val=<PostForm bound=False, valid=False, fields=(is_published;title;text;pub_date;location;category;image)>), PostModel = <class 'blog.models.Post'>
CommentModelAdapter = <class 'adapters.comment.CommentModelAdapter.<locals>._CommentModelAdapter'>, main_content_tester = <test_content.MainPostContentTester object at 0x0000018FC4E37B80>

    @pytest.mark.django_db(transaction=True)
    def test_post(
            published_category: Model,
            published_location: Model,
            user_client: django.test.Client,
            another_user_client: django.test.Client,
            unlogged_client: django.test.Client,
            comment_to_a_post: Model,
            create_post_context_form_item: Tuple[str, BaseForm],
            PostModel: Type[Model],
            CommentModelAdapter: CommentModelAdapterT,
            main_content_tester: MainPostContentTester
    ):
        _, ctx_form = create_post_context_form_item

        create_a_post_get_response = get_create_a_post_get_response_safely(
            user_client
        )

        response_on_created, created_items = _test_create_items(
            PostModel,
            PostModelAdapter,
            another_user_client,
            create_a_post_get_response,
            ctx_form,
            published_category,
            published_location,
            unlogged_client,
            user_client,
        )

        # checking images are visible on post creation
        created_content = response_on_created.content.decode('utf-8')
        img_count = created_content.count('<img')
        expected_img_count = main_content_tester.n_or_page_size(len(created_items))
        assert img_count >= expected_img_count, (
            'Убедитесь, что при создании публикации она отображается с картинкой.'
        )

        edit_response, edit_url, del_url = _test_edit_post(
            CommentModelAdapter,
            another_user_client,
            comment_to_a_post,
            unlogged_client=unlogged_client,
            user_client=user_client,
        )

        item_to_delete_adapter = PostModelAdapter(
            CommentModelAdapter(comment_to_a_post).post
        )
        del_url_addr = del_url.key

        del_unexisting_status_404_err_msg = (
            "Убедитесь, что при обращении к странице удаления "
            " несуществующего поста возвращается статус 404."
        )
        delete_tester = DeletePostTester(
            item_to_delete_adapter.item_cls,
            user_client,
            another_user_client,
            unlogged_client,
            item_adapter=item_to_delete_adapter,
        )
        delete_tester.test_delete_item(
            qs=item_to_delete_adapter.item_cls.objects.all(),
            delete_url_addr=del_url_addr,
        )
        try:
            AuthorisedSubmitTester(
                tester=delete_tester,
                test_response_cbk=SubmitTester.get_test_response_404_cbk(
                    err_msg=delete_tester.nonexistent_obj_error_message
                ),
            ).test_submit(url=del_url_addr, data={})
        except Post.DoesNotExist:
            raise AssertionError(del_unexisting_status_404_err_msg)

        err_msg_unexisting_status_404 = (
            "Убедитесь, что при обращении к странице "
            " несуществующего поста возвращается статус 404."
        )
        try:
            response = user_client.get(f"/posts/{item_to_delete_adapter.id}/")
            assert response.status_code == HTTPStatus.NOT_FOUND, (
                err_msg_unexisting_status_404)
        except Post.DoesNotExist:
            raise AssertionError(err_msg_unexisting_status_404)

        edit_status_code_not_404_err_msg = (
            "Убедитесь, что при обращении к странице редактирования"
            " несуществующего поста возвращается статус 404."
        )
        try:
            response = user_client.get(edit_url[0])
        except Post.DoesNotExist:
            raise AssertionError(edit_status_code_not_404_err_msg)

        assert response.status_code == HTTPStatus.NOT_FOUND, (
            edit_status_code_not_404_err_msg)

        @contextmanager
        def set_post_unpublished(post_adapter):
            is_published = post_adapter.is_published
            try:
                post_adapter.is_published = False
                post_adapter.save()
                yield
            finally:
                post_adapter.is_published = is_published
                post_adapter.save()

        @contextmanager
        def set_post_category_unpublished(post_adapter):
            category = post_adapter.category
            is_published = category.is_published
            try:
                category.is_published = False
                category.save()
                yield
            finally:
                category.is_published = is_published
                category.save()

        @contextmanager
        def set_post_postponed(post_adapter):
            pub_date = post_adapter.pub_date
            current_date = timezone.now()
            try:
                post_adapter.pub_date = post_adapter.pub_date.replace(
                    year=current_date.year + 1,
                    day=current_date.day - 1 or current_date.day)
                post_adapter.save()
                yield
            finally:
                post_adapter.pub_date = pub_date
                post_adapter.save()

        def check_post_access(client, post_adapter, err_msg, expected_status):
            url = f"/posts/{post_adapter.id}/"
            get_get_response_safely(client, url=url, err_msg=err_msg,
                                    expected_status=expected_status)

        # Checking unpublished post

        detail_post_adapter = PostModelAdapter(created_items[0])

        with set_post_unpublished(detail_post_adapter):
            check_post_access(
                user_client, detail_post_adapter,
                "Убедитесь, что страница поста, снятого с публикации, "
                "доступна автору этого поста.",
                expected_status=HTTPStatus.OK)
>           check_post_access(
                another_user_client, detail_post_adapter,
                "Убедитесь, что страница поста, снятого с публикации, "
                "доступна только автору этого поста.",
                expected_status=HTTPStatus.NOT_FOUND)

tests\test_post.py:224:
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ 
tests\test_post.py:211: in check_post_access
    get_get_response_safely(client, url=url, err_msg=err_msg,
tests\conftest.py:254: in get_get_response_safely
    response = user_client.get(url)
venv\lib\site-packages\django\test\client.py:742: in get
    response = super().get(path, data=data, secure=secure, **extra)
venv\lib\site-packages\django\test\client.py:396: in get
    return self.generic('GET', path, secure=secure, **{
venv\lib\site-packages\django\test\client.py:473: in generic
    return self.request(**r)
venv\lib\site-packages\django\test\client.py:719: in request
    self.check_exception(response)
venv\lib\site-packages\django\test\client.py:580: in check_exception
    raise exc_value
venv\lib\site-packages\django\core\handlers\exception.py:47: in inner
    response = get_response(request)
venv\lib\site-packages\django\core\handlers\base.py:204: in _get_response
    response = response.render()
venv\lib\site-packages\django\template\response.py:105: in render
    self.content = self.rendered_content
venv\lib\site-packages\django\template\response.py:83: in rendered_content
    return template.render(context, self._request)
venv\lib\site-packages\django\template\backends\django.py:61: in render
    return self.template.render(context)
venv\lib\site-packages\django\template\base.py:170: in render
    return self._render(context)
venv\lib\site-packages\django\test\utils.py:100: in instrumented_test_render
    return self.nodelist.render(context)
venv\lib\site-packages\django\template\base.py:938: in render
    bit = node.render_annotated(context)
venv\lib\site-packages\django\template\base.py:905: in render_annotated
    return self.render(context)
venv\lib\site-packages\django\template\loader_tags.py:150: in render
    return compiled_parent._render(context)
venv\lib\site-packages\django\test\utils.py:100: in instrumented_test_render
    return self.nodelist.render(context)
venv\lib\site-packages\django\template\base.py:938: in render
    bit = node.render_annotated(context)
venv\lib\site-packages\django\template\base.py:905: in render_annotated
    return self.render(context)
venv\lib\site-packages\django\template\loader_tags.py:62: in render
    result = block.nodelist.render(context)
venv\lib\site-packages\django\template\base.py:938: in render
    bit = node.render_annotated(context)
venv\lib\site-packages\django\template\base.py:905: in render_annotated
    return self.render(context)
venv\lib\site-packages\django\template\defaulttags.py:449: in render
    url = reverse(view_name, args=args, kwargs=kwargs, current_app=current_app)
venv\lib\site-packages\django\urls\base.py:86: in reverse
    return resolver._reverse_with_prefix(view, prefix, *args, **kwargs)
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ 

self = <URLResolver <module 'blog.urls' from 'D:\\Dev\\django_sprint4\\blogicum\\blog\\urls.py'> (blog:blog) ''>, lookup_view = 'profile', _prefix = '/', args = ('',), kwargs = {}
possibilities = [([('profile/%(username)s/', ['username'])], 'profile/(?P<username>[^/]+)/\\Z', {}, {'username': <django.urls.converters.StringConverter object at 0x0000018FC34D4C40>})]
possibility = [('profile/%(username)s/', ['username'])], pattern = 'profile/(?P<username>[^/]+)/\\Z', defaults = {}, converters = {'username': <django.urls.converters.StringConverter object at 0x0000018FC34D4C40>}
result = 'profile/%(username)s/', params = ['username'], candidate_subs = {'username': ''}, text_candidate_subs = {'username': ''}, match = True

    def _reverse_with_prefix(self, lookup_view, _prefix, *args, **kwargs):
        if args and kwargs:
            raise ValueError("Don't mix *args and **kwargs in call to reverse()!")

        if not self._populated:
            self._populate()

        possibilities = self.reverse_dict.getlist(lookup_view)

        for possibility, pattern, defaults, converters in possibilities:
            for result, params in possibility:
                if args:
                    if len(args) != len(params):
                        continue
                    candidate_subs = dict(zip(params, args))
                else:
                    if set(kwargs).symmetric_difference(params).difference(defaults):
                        continue
                    if any(kwargs.get(k, v) != v for k, v in defaults.items()):
                        continue
                    candidate_subs = kwargs
                # Convert the candidate subs to text using Converter.to_url().
                text_candidate_subs = {}
                match = True
                for k, v in candidate_subs.items():
                    if k in converters:
                        try:
                            text_candidate_subs[k] = converters[k].to_url(v)
                        except ValueError:
                            match = False
                            break
                    else:
                        text_candidate_subs[k] = str(v)
                if not match:
                    continue
                # WSGI provides decoded URLs, without %xx escapes, and the URL
                # resolver operates on such URLs. First substitute arguments
                # without quoting to build a decoded URL and look for a match.
                # Then, if we have a match, redo the substitution with quoted
                # arguments in order to return a properly encoded URL.
                candidate_pat = _prefix.replace('%', '%%') + result
                if re.search('^%s%s' % (re.escape(_prefix), pattern), candidate_pat % text_candidate_subs):
                    # safe characters from `pchar` definition of RFC 3986
                    url = quote(candidate_pat % text_candidate_subs, safe=RFC3986_SUBDELIMS + '/~:@')
                    # Don't allow construction of scheme relative urls.
                    return escape_leading_slashes(url)
        # lookup_view can be URL name or callable, but callables are not
        # friendly in error messages.
        m = getattr(lookup_view, '__module__', None)
        n = getattr(lookup_view, '__name__', None)
        if m is not None and n is not None:
            lookup_view_s = "%s.%s" % (m, n)
        else:
            lookup_view_s = lookup_view

        patterns = [pattern for (_, pattern, _, _) in possibilities]
        if patterns:
            if args:
                arg_msg = "arguments '%s'" % (args,)
            elif kwargs:
                arg_msg = "keyword arguments '%s'" % kwargs
            else:
                arg_msg = "no arguments"
            msg = (
                "Reverse for '%s' with %s not found. %d pattern(s) tried: %s" %
                (lookup_view_s, arg_msg, len(patterns), patterns)
            )
        else:
            msg = (
                "Reverse for '%(view)s' not found. '%(view)s' is not "
                "a valid view function or pattern name." % {'view': lookup_view_s}
            )
>       raise NoReverseMatch(msg)
E       django.urls.exceptions.NoReverseMatch: Reverse for 'profile' with arguments '('',)' not found. 1 pattern(s) tried: ['profile/(?P<username>[^/]+)/\\Z']

venv\lib\site-packages\django\urls\resolvers.py:698: NoReverseMatch
