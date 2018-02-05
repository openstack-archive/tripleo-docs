Unit tests
----------

Now, let’s continue writing unit tests for the Mistral action we
wrote before. Let’s add the
**tripleo_common/tests/actions/test_undercloud.py** file with the
following content in the **tripleo-comon** repository.

::

    import mock

    from tripleo_common.actions import undercloud
    from tripleo_common.tests import base


    class GetFreeSpaceTest(base.TestCase):
        def setUp(self):
            super(GetFreeSpaceTest, self).setUp()
            self.temp_dir = "/tmp"

        @mock.patch('tempfile.gettempdir')
        @mock.patch("os.path.isdir")
        @mock.patch("os.statvfs")
        def test_run_false(self, mock_statvfs, mock_isdir, mock_gettempdir):
            mock_gettempdir.return_value = self.temp_dir
            mock_isdir.return_value = True
            mock_statvfs.return_value = mock.MagicMock(
                spec_set=['f_frsize', 'f_bavail'],
                f_frsize=4096, f_bavail=1024)
            action = undercloud.GetFreeSpace()
            action_result = action.run(context={})
            mock_gettempdir.assert_called()
            mock_isdir.assert_called()
            mock_statvfs.assert_called()
            self.assertEqual("There is no enough space, avail. - 4 MB",
                             action_result.error['msg'])

        @mock.patch('tempfile.gettempdir')
        @mock.patch("os.path.isdir")
        @mock.patch("os.statvfs")
        def test_run_true(self, mock_statvfs, mock_isdir, mock_gettempdir):
            mock_gettempdir.return_value = self.temp_dir
            mock_isdir.return_value = True
            mock_statvfs.return_value = mock.MagicMock(
                spec_set=['f_frsize', 'f_bavail'],
                f_frsize=4096, f_bavail=10240000)
            action = undercloud.GetFreeSpace()
            action_result = action.run(context={})
            mock_gettempdir.assert_called()
            mock_isdir.assert_called()
            mock_statvfs.assert_called()
            self.assertEqual("There is enough space, avail. - 40000 MB",
                             action_result.data['msg'])

Run

::

    tox -epy27

to see any unit tests errors.
