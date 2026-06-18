# Changes Made to sim

## run_spike_dev.sh
```sh
# -- Changed --
ROS_DOMAIN_ID="${ROS_DOMAIN_ID:-2}"
DOCKER_ARGS+=("-e RMW_IMPLEMENTATION=${RMW_IMPLEMENTATION:-rmw_cyclonedds_cpp}")
```

## sim_spike_launch.py
### Header
```sh
# -- Added --
from launch.actions import (
    DeclareLaunchArgument,
    ExecuteProcess,
    GroupAction,
    IncludeLaunchDescription,
    OpaqueFunction,
    TimerAction,  # Added
)

# -- Changed --
from launch_ros.actions import Node, PushRosNamespace
```
### Perception Pipeline
The multi-cam feature works perfectly fine but is currently not being implemented in testing due to GPU restrictions. When running on the proper GPUs this code will work fine (it's been tested with two cameras and worked great but GPU almost crashed). Add launch argument declarations below as well for future use.
```sh
# -- Added --
# ************ Multi-cam ************
    # perception_pkg = get_package_share_directory('spike_pipeline')
    # pipeline_config_path = os.path.join(
    #     perception_pkg, 
    #     'config', 
    #     'motion_rfdetr_botsort_pipeline.yaml'
    # )
    # perception_debug = LaunchConfiguration('perception_debug').perform(context)
    # cam_indices = [int(c) for c in
    #     LaunchConfiguration('cam_indices').perform(context).split(',') if c.strip()]
    # stagger = float(LaunchConfiguration('perception_stagger_sec').perform(context))

    # for n, cam in enumerate(cam_indices):
    #     node_name = 'motion_rfdetr_botsort' if n == 0 else f'motion_rfdetr_botsort_cam{cam}'
    #     actions.append(TimerAction(period=n * stagger, actions=[Node(
    #         package='spike_pipeline', executable='motion_rfdetr_botsort_node',
    #         name=node_name, namespace=namespace, output='screen',
    #         parameters=[{
    #             'use_sim_time': True, 'input_source_type': 'ros_topic',
    #             'image_topic': f'/{namespace}/cam_{cam}/image_raw',
    #             'frame_id': f'cam_{cam}_optical',
    #             'pipeline_config_path': pipeline_config_path,
    #             'publish_detection_topics': True, 'perception_debug': perception_debug,
    #             'use_reliable_qos': True, 'input_timeout_ms': 5000, 'input_queue_size': 30,
    #         }])]))
```
**Only if using multi-cam, delete the following code**
```sh
# -- Deleted --
# ************ Single-cam ************
    actions.append(Node(
        package='foxglove_bridge', executable='foxglove_bridge',
        name='foxglove_bridge', namespace=namespace, output='screen',
        parameters=[{'use_sim_time': True, 'port': 8765, 'address': '0.0.0.0',
                     'send_buffer_limit': 10000000, 'use_compression': True}]))
    perception_pkg = get_package_share_directory('spike_pipeline')
    perception_launch = GroupAction([
        PushRosNamespace(namespace),
        IncludeLaunchDescription(
            PythonLaunchDescriptionSource(
                os.path.join(perception_pkg, 'launch', 'isaac_sim_perception.launch.py')
            ),
            launch_arguments={
                'camera_topic': f'/{namespace}/cam_0/image_raw',
                'perception_debug': LaunchConfiguration('perception_debug').perform(context),
            }.items()
        ),
    ])
```
### Scout launch args
```sh
# -- Added --
'enable_ego_motion':'false'
'enable_ins_gps_relative':'false'
'canonical_imu_topic': f'/{namespace}/imuu/data',
```
### Hangar launch args
```sh
# -- Added --
'params_file':os.path.join(hangar_pkg, 'config', 'field_hangar_1.yaml'),
```
### Launch Description 
```sh
# -- Added --
### For multi-camera
DeclareLaunchArgument('cam_indices', default_value='0', description='Comma-separated camera indices, e.g. "0,1,2,3,4"'),
DeclareLaunchArgument('perception_stagger_sec', default_value='8.0', description='Delay (s) between per-camera TRT engine loads'),
```

## spawn_spotter.launch.py
### Header
```sh
# -- Added --
import os
import xacro
from ament_index_python.packages import get_package_share_directory
from launch.actions import (
    DeclareLaunchArgument, 
    IncludeLaunchDescription,
    LogInfo,
    OpaqueFunction # Added here
from launch_ros.actions import Node
```
### Body
#### Launch Setup --> OpaqueFunction parameter
```sh
def launch_setup(context, *args, **kwargs):
    pkg_share = FindPackageShare('spike_sim')
    namespace = LaunchConfiguration('namespace').perform(context)

    actions = []

    # Include base spawn_robot with spotter configuration
    actions.append(IncludeLaunchDescription(
        PythonLaunchDescriptionSource(
            PathJoinSubstitution([pkg_share, 'launch', 'spawn_robot.launch.py'])
        ),
        launch_arguments={
            'robot': 'spotter',
            'side_lens': LaunchConfiguration('side_lens'),
            'high_fidelity': LaunchConfiguration('high_fidelity'),
            'namespace': LaunchConfiguration('namespace'),
            'x': LaunchConfiguration('x'),
            'y': LaunchConfiguration('y'),
            'z': LaunchConfiguration('z'),
            'roll': LaunchConfiguration('roll'),
            'pitch': LaunchConfiguration('pitch'),
            'yaw': LaunchConfiguration('yaw'),
            'fixed': LaunchConfiguration('fixed'),
            'api_host':LaunchConfiguration('api_host'),
            'api_port': LaunchConfiguration('api_port'),
            'spike_ws_path': LaunchConfiguration('spike_ws_path'),
        }.items()
    ))
    if not namespace:
        actions.append(LogInfo(
            msg='spawn_spotter: namespace is empty - skipping robot_state_publisher '
                'and the base_link TF bridge. Pass namespace:=<robot> (e.g. '
                'spotter_0) to enable spotter TF for scout.'
        ))
        return actions

    desc_share = get_package_share_directory('spike_description')
    spotter_urdf = os.path.join(desc_share, 'urdf', 'quad_spotter.urdf.xacro')
    robot_description = xacro.process_file(spotter_urdf).toxml()

    actions.append(Node(
        package='robot_state_publisher',
        executable='robot_state_publisher',
        name='spotter_state_publisher',
        output='screen',
        parameters=[{
            'robot_description': robot_description,
            'use_sim_time': True,
            'frame_prefix': '',
        }],
    ))

    actions.append(Node(
        package='tf2_ros',
        executable='static_transform_publisher',
        name='spotter_root_to_base_link',
        output='screen',
        parameters=[{'use_sim_time': True}],
        arguments=['--x', '0', '--y', '0', '--z', '0',
                    '--yaw', '0', '--pitch', '0', '--roll', '0',
                   '--frame-id', namespace, '--child-frame-id', 'base_link'],
    ))

    return actions
```
#### OpaqueFunction in generate_launch_description
```sh
# Removed 
IncludeLaunchDescription(...)

# Added 
DeclareLaunchArgument('api_port', default_value='8080', description='REST API port'),
        DeclareLaunchArgument('spike_ws_path', default_value='', description='Override path to spike_ws (native mode only)'),

OpaqueFunction(function=launch_setup) # Added 
```

## scout_sim.yaml
If left unchanged, `/scout_3d_transform/spatial_detections` and `/scout_3d_transform/tracks` will output empty detections. However, `.../..._all` will output unfiltered sensor / ground truth detections. If you apply these changes, you bypass the sensor gates making `/spatial_detections == /spatial_detections_all`  and `/tracks == /tracks_all`. This is a quick hack to make detections topic output something; however, this is not how it is done on the real platform.
```sh
# Changed
enable_self_distance_filter: false

# Added
enable_spotter_motion_gate: false
enable_moving_filter: false
coast_publish_max_stale_sec: 0.0
```

## Git LFS fetch of .plan files for RTX-40xx
If you are using a 50xx GPU this step is not necessary. 
- Detector: `rfdetr_560_finetuned_4.plan`
- BotSORT ReID: `osnet_x0_25_market1501.plan`
- STARK encoder: `backbone_bottleneck_pe.plan`
- STARK head: `complete.plan`

