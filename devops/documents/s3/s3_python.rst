S3 in Python
============

Open a Connection
-----------------

.. code:: python

    import boto
    connection = boto.connect_s3(
        host="objects.rma.cloudscale.ch",
        aws_access_key_id="ACCESS_KEY",
        aws_secret_access_key="SECRET_KEY")

``ACCESS_KEY`` and ``SECRET_KEY`` are the credentials.


List Buckets
------------

    .. code:: python

        buckets = connection.get_all_buckets()


Get a Bucket
------------

    .. code:: python

        bucket = connection.get("BUCKET_NAME")


Get an Object
-------------

    .. code:: python

        key = bucket.get("KEY")

    ``KEY`` is the sha2 hash of the object. It corresponds to the column ``_nice_binary.hash`` in Nice.

    * Now that you have the ``key``, you can load the object into memory

        .. code:: python

            binary_content = key.get_contents_as_string()

    * … download it to a file

        .. code:: python

            key.get_contents_to_filename('PATH/TO/FILE')

    * … or create a link for download

        Link valid for 3600s:

        .. code:: python

            url = k.generate_url(3600)

        Link valid for 3600s with custom value for response header ``Content-Type``:

        .. code:: python

            url = key.generate_url(3600, response_headers={'response-content-type': 'text/plain'})


