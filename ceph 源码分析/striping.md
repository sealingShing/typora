# striping

```shell
[root@source1 wfs_test]# rbd create test/test_tripe_3 --stripe-unit 4KB --stripe-count=3 --size 10G
2021-07-12 14:43:32.770 400259dd0010 10 librbd: create name=test_tripe_3, id= 85606b8b4567, size=10737418240, opts=[format=2, stripe_unit=4096, stripe_count=3]
2021-07-12 14:43:32.780 400259dd0010 20 librbd::image::CreateRequest: 0xaaae91693bd0 CreateRequest: name=test_tripe_3, id=85606b8b4567, size=10737418240, features=63, order=22, stripe_unit=4096, stripe_count=3, journal_order=24, journal_splay_width=4, journal_pool=, data_pool=
2021-07-12 14:43:32.780 400259dd0010 20 librbd::image::CreateRequest: 0xaaae91693bd0 send: 
2021-07-12 14:43:32.780 400259dd0010 20 librbd::image::CreateRequest: 0xaaae91693bd0 validate_data_pool: 
2021-07-12 14:43:32.780 400259dd0010  5 librbd::image::ValidatePoolRequest: read_rbd_info: 
2021-07-12 14:43:32.780 40025be9ae00  5 librbd::image::ValidatePoolRequest: handle_read_rbd_info: r=0
2021-07-12 14:43:32.780 40025be9ae00  5 librbd::image::ValidatePoolRequest: finish: r=0
2021-07-12 14:43:32.780 40025be9ae00 20 librbd::image::CreateRequest: 0xaaae91693bd0 handle_validate_data_pool: r=0
2021-07-12 14:43:32.780 40025be9ae00 20 librbd::image::CreateRequest: 0xaaae91693bd0 create_id_object: 
2021-07-12 14:43:32.790 40025be9ae00 20 librbd::image::CreateRequest: 0xaaae91693bd0 handle_create_id_object: r=0
2021-07-12 14:43:32.790 40025be9ae00 20 librbd::image::CreateRequest: 0xaaae91693bd0 add_image_to_directory: 
2021-07-12 14:43:32.790 40025be9ae00 20 librbd::image::CreateRequest: 0xaaae91693bd0 handle_add_image_to_directory: r=0
2021-07-12 14:43:32.790 40025be9ae00 20 librbd::image::CreateRequest: 0xaaae91693bd0 negotiate_features: 
2021-07-12 14:43:32.790 40025be9ae00 20 librbd::image::CreateRequest: 0xaaae91693bd0 handle_negotiate_features: r=0
2021-07-12 14:43:32.790 40025be9ae00 20 librbd::image::CreateRequest: 0xaaae91693bd0 create_image: 
2021-07-12 14:43:32.790 40025be9ae00 20 librbd::image::CreateRequest: 0xaaae91693bd0 handle_create_image: r=0
2021-07-12 14:43:32.790 40025be9ae00 20 librbd::image::CreateRequest: 0xaaae91693bd0 set_stripe_unit_count: 
2021-07-12 14:43:32.790 40025be9ae00 20 librbd::image::CreateRequest: 0xaaae91693bd0 handle_set_stripe_unit_count: r=0
2021-07-12 14:43:32.790 40025be9ae00 20 librbd::image::CreateRequest: 0xaaae91693bd0 object_map_resize: 
2021-07-12 14:43:32.800 40025be9ae00 20 librbd::image::CreateRequest: 0xaaae91693bd0 handle_object_map_resize: r=0
2021-07-12 14:43:32.800 40025be9ae00 20 librbd::image::CreateRequest: 0xaaae91693bd0 complete: 
2021-07-12 14:43:32.800 40025be9ae00 20 librbd::image::CreateRequest: 0xaaae91693bd0 complete: done.

```





```shell
[root@host123 ~]# rbd create volumes/test_stripe_5 --stripe-unit=4KB --stripe-count=5 --size 50G
2021-07-12 15:07:31.740 400143ec0010 10 librbd: create name=test_stripe_5, id= 2a92f6b8b4567, size=53687091200, opts=[format=2, stripe_unit=4096, stripe_count=5]
2021-07-12 15:07:31.740 400143ec0010 20 librbd::image::CreateRequest: 0xaaab6e053be0 CreateRequest: name=test_stripe_5, id=2a92f6b8b4567, size=53687091200, features=63, order=22, stripe_unit=4096, stripe_count=5, journal_order=24, journal_splay_width=4, journal_pool=, data_pool=
2021-07-12 15:07:31.740 400143ec0010 20 librbd::image::CreateRequest: 0xaaab6e053be0 send: 
2021-07-12 15:07:31.740 400143ec0010 20 librbd::image::CreateRequest: 0xaaab6e053be0 validate_data_pool: 
2021-07-12 15:07:31.740 400143ec0010  5 librbd::image::ValidatePoolRequest: read_rbd_info: 
2021-07-12 15:07:31.750 400145f8ae00  5 librbd::image::ValidatePoolRequest: handle_read_rbd_info: r=0
2021-07-12 15:07:31.750 400145f8ae00  5 librbd::image::ValidatePoolRequest: finish: r=0
2021-07-12 15:07:31.750 400145f8ae00 20 librbd::image::CreateRequest: 0xaaab6e053be0 handle_validate_data_pool: r=0
2021-07-12 15:07:31.750 400145f8ae00 20 librbd::image::CreateRequest: 0xaaab6e053be0 create_id_object: 
2021-07-12 15:07:31.750 400145f8ae00 20 librbd::image::CreateRequest: 0xaaab6e053be0 handle_create_id_object: r=0
2021-07-12 15:07:31.750 400145f8ae00 20 librbd::image::CreateRequest: 0xaaab6e053be0 add_image_to_directory: 
2021-07-12 15:07:31.750 400145f8ae00 20 librbd::image::CreateRequest: 0xaaab6e053be0 handle_add_image_to_directory: r=0
2021-07-12 15:07:31.750 400145f8ae00 20 librbd::image::CreateRequest: 0xaaab6e053be0 negotiate_features: 
2021-07-12 15:07:31.750 400145f8ae00 20 librbd::image::CreateRequest: 0xaaab6e053be0 handle_negotiate_features: r=0
2021-07-12 15:07:31.750 400145f8ae00 20 librbd::image::CreateRequest: 0xaaab6e053be0 create_image: 
2021-07-12 15:07:31.750 400145f8ae00 20 librbd::image::CreateRequest: 0xaaab6e053be0 handle_create_image: r=0
2021-07-12 15:07:31.750 400145f8ae00 20 librbd::image::CreateRequest: 0xaaab6e053be0 set_stripe_unit_count: 
2021-07-12 15:07:31.750 400145f8ae00 20 librbd::image::CreateRequest: 0xaaab6e053be0 handle_set_stripe_unit_count: r=0
2021-07-12 15:07:31.750 400145f8ae00 20 librbd::image::CreateRequest: 0xaaab6e053be0 object_map_resize: 
2021-07-12 15:07:31.760 400145f8ae00 20 librbd::image::CreateRequest: 0xaaab6e053be0 handle_object_map_resize: r=0
2021-07-12 15:07:31.760 400145f8ae00 20 librbd::image::CreateRequest: 0xaaab6e053be0 complete: 
2021-07-12 15:07:31.760 400145f8ae00 20 librbd::image::CreateRequest: 0xaaab6e053be0 complete: done.
```

