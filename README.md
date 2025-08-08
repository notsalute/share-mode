# share-mode

	void HandleThirdPersonCamera(Unity::CGameObject* LocalPlayerGameObject) {
		std::lock_guard lock(ThirdPersonMutex);

		if (!LocalPlayerGameObject) return;

		if (!SetupComplete) {
			auto* current_camera = reinterpret_cast<Unity::CComponent*>(Unity::Camera::GetCurrent());
			if (!current_camera) return;

			MainCameraObject = current_camera->GetGameObject();
			if (!MainCameraObject) return;

			MainCameraTransform = MainCameraObject->GetTransform();
			if (!MainCameraTransform) return;

			auto create_third_person_camera = [](Unity::Vector3 rotation, float distance, Unity::CTransform* parent) -> Unity::CComponent* {
				if (!parent) return nullptr;

				auto cam_object = Unity::GameObject::CreatePrimitive(Unity::GameObject::m_ePrimitiveType::Cube);
				if (!cam_object) return nullptr;

				auto* renderer = cam_object->GetComponent("MeshRenderer");
				if (renderer) renderer->SetPropertyValue<bool>("enabled", false);

				auto* transform = cam_object->GetTransform();
				if (!transform) return nullptr;

				transform->SetLocalScale(parent->GetLocalScale());
				transform->SetPropertyValue<Unity::CTransform*>("parent", parent);
				transform->SetRotation(parent->GetRotation());
				transform->CallMethodSafe<void*>("Rotate", rotation, 1);

				auto forward = transform->GetPropertyValue<Unity::Vector3>("forward");
				auto pos = parent->GetPosition();

				transform->SetPosition(Unity::Vector3(
					pos.x + forward.x * -distance,
					pos.y + forward.y * -distance,
					pos.z + forward.z * -distance
				));

				auto cam_type = IL2CPP::Class::GetSystemType(IL2CPP::Class::Find(hash_str("UnityEngine.Camera")));
				if (!cam_type) return nullptr;

				auto* camera = cam_object->AddComponent(cam_type);
				if (!camera) return nullptr;

				camera->CallMethodSafe<void*>(hash_str("set_nearClipPlane"), 0.004f);
				camera->SetPropertyValue<bool>(hash_str("enabled"), false);
				cam_object->DontDestroyOnload();

				return camera;
				};

			BackCamera = create_third_person_camera(Unity::Vector3(0, 0, 0), 2.5f, MainCameraTransform);
			FrontCamera = create_third_person_camera(Unity::Vector3(0, 180, 0), 2.5f, MainCameraTransform);
			SetupComplete = true;
		}

		if (!ThirdPersonActive) {
			if (Settings::Exploits::ThirdPersonMode == 0 && FrontCamera) {
				FrontCamera->SetPropertyValue<bool>(hash_str("enabled"), true);
				if (BackCamera) BackCamera->SetPropertyValue<bool>(hash_str("enabled"), false);
			}
			else if (Settings::Exploits::ThirdPersonMode == 1) {
				BackCamera->SetPropertyValue<bool>(hash_str("enabled"), true);
				if (FrontCamera) FrontCamera->SetPropertyValue<bool>(hash_str("enabled"), false);
			}
			ThirdPersonActive = true;
		}
	}

	void DisableThirdPersonCamera() {
		if (!ThirdPersonActive) return;

		if (FrontCamera) FrontCamera->SetPropertyValue<bool>(hash_str("enabled"), false);
		if (BackCamera) BackCamera->SetPropertyValue<bool>(hash_str("enabled"), false);

		auto* current_camera = reinterpret_cast<Unity::CComponent*>(Unity::Camera::GetCurrent());
		if (current_camera) {
			current_camera->SetPropertyValue<bool>(hash_str("enabled"), true);

			auto* cam_tf = current_camera->GetGameObject()->GetTransform();
			if (cam_tf && MainCameraTransform) {
				cam_tf->SetPosition(MainCameraTransform->GetPosition());
				cam_tf->SetRotation(MainCameraTransform->GetRotation());
			}
		}

		ThirdPersonActive = false;
	}
